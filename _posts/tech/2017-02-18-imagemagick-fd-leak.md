---
title: imagemagick文件句柄泄露分析
layout: post
categories: 技术
tags:
- imageMagick
comments: true
description: 记录实际的生产环境中遇到的BUG及解决问题的一些经验
---


# 1.问题发现
博主负责的一个在线图片处理服务，存在长时间（超过20天）运行后会服务进程出错退出的问题。运维小伙伴把情况报到开发这，并没有更多的信息😵😵😵

# 2.初步分析
## 2.1.产品服务
该产品是一个云计算图片处理服务（参见七牛数据处理服务），后台使用到了imageMagick SDK的基本图片处理功能。

服务同时部署在物理服务器和容器中，多个异地区域机房，但实际仅在其中一个机房的物理集群中发生进程重启的情况（服务使用supervisorctl托管，异常退出后自动重启）。

线上服务并未开启coredump；出core日志调用栈仅显示捕获signal 11，回放上一条的服务请求并不能触发错误。

查看该服务在此DC中监控情况，发现峰值TPS在40K+，日均TPS约在20K左右，CPU、内存、磁盘和网络带宽均在警戒值以下，服务器句柄数保持在最高维持在10K左右。

## 2.2.诊断
根据经验，长时间运行后服务进程出错退出，往往是因为存在资源泄露，例如内存泄露、句柄泄露、临时文件未删除等。

调取服务进程的物理资源使用情况进行分析，CPU、内存均无显著异常，但文件句柄使用情况存在问题

![服务句柄量](http://ohgqsah1g.bkt.clouddn.com/20170218/fd-leak.png)

从上图中可以看出，服务句柄数基本呈线性增长，陡降的地方为服务进程被重启了（比如服务更新发布），这对于一个长期运行的服务端程序是不合理的。

故能初步判断出，程序存在文件句柄泄露。

# 3.深入分析
## 3.1.线上分析
该服务部署在多个DC，服务发生进程重启均集中在一个DC。该DC服务请求最高，峰值TPS可以达到30+K；相对其他DC的请求量要小一个数量级。

登录服务部署在DC的一台服务器，使用lsof查询服务进行FD使用情况，可以发现存在大量的FD处于deleted状态（标记删除但仍未释放）。

```
lsof -p 9291  | grep deleted

...
qboximage 9291 qboxserver 1228r      REG        8,1     31346    4487983 /home/qboxserver/image6/_package/run/fopd_tmpdir/mogr2_244b_34d98ac907d245_452020_1d0 (deleted)
qboximage 9291 qboxserver 1229r      REG        8,1     18542    4487917 /home/qboxserver/image6/_package/run/fopd_tmpdir/mogr2_244b_34d9871a4080c3_44d568_364 (deleted)
qboximage 9291 qboxserver 1230r      REG        8,1     37602    4488004 /home/qboxserver/image6/_package/run/fopd_tmpdir/mogr2_244b_34d9858c894d31_44b62d_25f (deleted)
qboximage 9291 qboxserver 1231r      REG        8,1     50062    4487872 /home/qboxserver/image6/_package/run/fopd_tmpdir/mogr2_244b_34d97fe5e79906_44401e_26d (deleted)
qboximage 9291 qboxserver 1232r      REG        8,1     12640    4487898 /home/qboxserver/image6/_package/run/fopd_tmpdir/mogr2_244b_34d9894b49bfcc_45018b_2d9 (deleted)
qboximage 9291 qboxserver 1233r      REG        8,1     49844    4488020 /home/qboxserver/image6/_package/run/fopd_tmpdir/mogr2_244b_34d989f6040b2d_450f17_2bf (deleted)
qboximage 9291 qboxserver 1239r      REG        8,1     10772    4487929 /home/qboxserver/image6/_package/run/fopd_tmpdir/mogr2_244b_34d980dc71106c_4454e1_1ef (deleted)
qboximage 9291 qboxserver 1240r      REG        8,1     73538    4487977 /home/qboxserver/image6/_package/run/fopd_tmpdir/mogr2_244b_34d989d52655ea_450c7e_1b5 (deleted)
...
```
进程重启后，单日可堆积未释放的FD到500个左右

## 3.2.日志分析
用lsof查看未释放的FD，发现都是指向服务流程中的临时文件，文件路径中包含唯一HASH字符串。

在日志系统中检索上述hash值，发现都是同一种请求类型，即webp图片转成webp请求，猜测可能与webp处理例程有关。

随机抓取几个请求，任选一台服务器，进行请求回访，结合日志系统使用lsof检索，发现确实每一个webp转webp的请求都会产生一个FD泄露。构造类似的请求，发送线上其他DC进行处理，通过后台日志+lsof，发现同样也发生了FD泄露。

抓取线上文件，将线上服务请求在本地开发环境进行回放，同样也可以产生句柄泄露。

至此，BUG可以稳定复现，可以确定针对webp转webp的请求存在句柄泄露。

# 4.调试分析
## 4.1.泄露定位
那么问题来了，泄露发生在什么地方呢？该服务端http server使用go语言，图片处理使用cgo调用开源库imageMagick的SDK，服务代码就有上千行，开源库部分博主还不够熟悉😳😳😳

很显然，生啃代码去找BUG绝对不是个好主意，需要另想办法。

简单浏览imageMagick的代码，可以发现大部分都是C语言写的（Magick++部分进行了面向对象的封装，使用了C++）。感谢男神谭浩强，博主还记得，C语言中涉及文件句柄的函数主要有open、close、fopen和fclose，也是想到了可以通过Hack底层libc的库函数。

废话不多说，上代码
```
#include <stdio.h>
#include <string.h>
#include <dlfcn.h>
#include <execinfo.h>

typedef int(*OPEN) (const char * pathname, int flags);
typedef int(*CLOSE) (int fildes);
typedef FILE*(*FOPEN) (const char * path, const char * mode);
typedef int(*FCLOSE)(FILE* stream);


/* 打印调用栈的最大深度 */
#define DUMP_STACK_DEPTH_MAX 16
void dump_trace() {
    void *stack_trace[DUMP_STACK_DEPTH_MAX] = {0};
    char **stack_strings = NULL;
    int stack_depth = 0;
    int i = 0;

    /* 获取栈中各层调用函数地址 */
    stack_depth = backtrace(stack_trace, DUMP_STACK_DEPTH_MAX);

    /* 查找符号表将函数调用地址转换为函数名称 */
    stack_strings = (char **)backtrace_symbols(stack_trace, stack_depth);
    if (NULL == stack_strings) {
        printf(" Memory is not enough while dump Stack Trace! \r\n");
        return;
    }

    /* 打印调用栈 */
    printf(" Stack Trace: \r\n");
    for (i = 0; i < stack_depth; ++i) {
        printf(" [%d] %s \r\n", i, stack_strings[i]);
    }

    /* 获取函数名称时申请的内存需要自行释放 */
    free(stack_strings);
    stack_strings = NULL;

    return;
}

int open(const char * pathname, int flags)
{
    static void *handle = NULL;
    static OPEN old_open = NULL;
    if( !handle )
    {
        handle = dlopen("libc.so.6", RTLD_LAZY);
        old_open = (OPEN)dlsym(handle, "open");
    }
    int fd = old_open(pathname, flags); 
    printf("open function invoked. path=[%s] flag=<%d> fd=[%d]\n", pathname, flags, fd);
    dump_trace();
    return fd;
}

int close(int fildes)
{
    static void *handle = NULL;
    static CLOSE old_close = NULL;
    if( !handle )
    {
        handle = dlopen("libc.so.6", RTLD_LAZY);
        old_close = (CLOSE)dlsym(handle, "open");
    }
    printf("close function invoked. fd=[%d] flag=<%d>\n", fildes);
    dump_trace();
    return old_close(fildes);
}

FILE* fopen(const char * path, const char * mode)
{
    static void *handle = NULL;
    static FOPEN old_fopen = NULL;
    if( !handle )
    {
        handle = dlopen("libc.so.6", RTLD_LAZY);
        old_fopen = (FOPEN)dlsym(handle, "fopen");
    }
    FILE* f =  old_fopen(path,mode);
    printf("fopen function invoked. path=[%s] flag=<%s> FILE=[%p]\n", path, mode,f);
    dump_trace();
    return f;
}

int fclose(FILE* stream)
{
    static void *handle = NULL;
    static FCLOSE old_fclose = NULL;
    if( !handle )
    {
        handle = dlopen("libc.so.6", RTLD_LAZY);
        old_fclose = (FCLOSE)dlsym(handle, "fclose");
    }
    printf("fclose function invoked. FILE=[%p]\n", stream);
    dump_trace();
    return old_fclose(stream);
}
```
上述代码勾了libc中的open、close、fopen和fclose函数，打印出了调用参数，以及函数调用堆栈

编译成动态链接库
```
gcc -fPIC -shared -o hook.so hook.c -ldl
```

手动挂载动态链接库，本地启动服务程序
```
LD_LIBRARY_PATH=./hook.so ./qboximage -f ./qboximage.conf
```

重新发送之前的请求，可以从日志中获取对应FD的情况如下：
第一次句柄打开(fopen)
```
fopen function invoked. path=[/workspace/tmp/mogr2_7_34d6663757bf64_1_31a] flag=<rb> FILE=[0x7fe4b403a5b0]
 Stack Trace: 

 [0] /root/hook.so(dump_trace+0x52) [0x7fe4c8e609a7] 

 [1] /root/hook.so(fopen+0x96) [0x7fe4c8e60c07] 

 [2] /root/fop_depend/_package/lib_for_fopd/libMagickCore-6.Q16.so.2(OpenBlob+0x6eb) [0x7fe4c85112eb] 

 [3] /root/fop_depend/_package/lib_for_fopd/libMagickCore-6.Q16.so.2(SetImageInfo+0x262) [0x7fe4c85d8332] 

 [4] /root/fop_depend/_package/lib_for_fopd/libMagickCore-6.Q16.so.2(ReadImage+0xa1) [0x7fe4c85454a1] 

 [5] /root/fop_depend/_package/lib_for_fopd/libMagickCore-6.Q16.so.2(ReadImages+0x15b) [0x7fe4c85465db] 

 [6] /root/fop_depend/_package/lib_for_fopd/libMagickWand-6.Q16.so.2(ConvertImageCommand+0x993) [0x7fe4c8b83d53] 

 [7] /root/fop_depend/_package/lib_for_fopd/libMagickWand-6.Q16.so.2(MagickCommandGenesis+0x6b3) [0x7fe4c8bf0b63] 

 [8] /root/dora-image(_cgo_f771dc2ebe72_Cfunc_myConvert+0x4b) [0x86cd5b] 

 [9] /root/dora-image() [0x475770] 
```
对应的句柄关闭：
```
fclose function invoked. FILE=[0x7fe4b403a5b0]
 Stack Trace: 

 [0] /root/hook.so(dump_trace+0x52) [0x7fe4c8e609a7] 

 [1] /root/hook.so(fclose+0x6f) [0x7fe4c8e60c7c] 

 [2] /root/fop_depend/_package/lib_for_fopd/libMagickCore-6.Q16.so.2(CloseBlob+0x241) [0x7fe4c8510b21] 

 [3] /root/fop_depend/_package/lib_for_fopd/libMagickCore-6.Q16.so.2(SetImageInfo+0x343) [0x7fe4c85d8413] 

 [4] /root/fop_depend/_package/lib_for_fopd/libMagickCore-6.Q16.so.2(ReadImage+0xa1) [0x7fe4c85454a1] 

 [5] /root/fop_depend/_package/lib_for_fopd/libMagickCore-6.Q16.so.2(ReadImages+0x15b) [0x7fe4c85465db] 

 [6] /root/fop_depend/_package/lib_for_fopd/libMagickWand-6.Q16.so.2(ConvertImageCommand+0x993) [0x7fe4c8b83d53] 

 [7] /root/fop_depend/_package/lib_for_fopd/libMagickWand-6.Q16.so.2(MagickCommandGenesis+0x6b3) [0x7fe4c8bf0b63] 

 [8] /root/dora-image(_cgo_f771dc2ebe72_Cfunc_myConvert+0x4b) [0x86cd5b] 

 [9] /root/dora-image() [0x475770] 
```
第二次句柄打开：
```
fopen function invoked. path=[/workspace/tmp/mogr2_7_34d6663757bf64_1_31a] flag=<rb> FILE=[0x7fe4b403de50]
 Stack Trace: 

 [0] /root/hook.so(dump_trace+0x52) [0x7fe4c8e609a7] 

 [1] /root/hook.so(fopen+0x96) [0x7fe4c8e60c07] 

 [2] /root/fop_depend/_package/lib_for_fopd/libMagickCore-6.Q16.so.2(OpenBlob+0x6eb) [0x7fe4c85112eb] 

 [3] /root/fop_depend/_package/lib_for_fopd/libMagickCore-6.Q16.so.2(+0x30aada) [0x7fe4c8791ada] 

 [4] /root/fop_depend/_package/lib_for_fopd/libMagickCore-6.Q16.so.2(ReadImage+0x198) [0x7fe4c8545598] 

 [5] /root/fop_depend/_package/lib_for_fopd/libMagickCore-6.Q16.so.2(ReadImages+0x15b) [0x7fe4c85465db] 

 [6] /root/fop_depend/_package/lib_for_fopd/libMagickWand-6.Q16.so.2(ConvertImageCommand+0x993) [0x7fe4c8b83d53] 

 [7] /root/fop_depend/_package/lib_for_fopd/libMagickWand-6.Q16.so.2(MagickCommandGenesis+0x6b3) [0x7fe4c8bf0b63] 

 [8] /root/dora-image(_cgo_f771dc2ebe72_Cfunc_myConvert+0x4b) [0x86cd5b] 

 [9] /root/dora-image() [0x475770] 
```
然后就么有关闭了。。。好吧，至此确定了FD在这里落跑了

## 代码分析
那么问题又来了，根据调用堆栈，可以看到第二次调用fopen的是OpenBlob函数。grep了一把imageMagick的代码，发现有好几十处。。。博主最恨这种大海捞针的看代码😠，一定要再想想办法

既然有了调用堆栈，也就有了函数起始地址和调用偏移地址，为啥不反汇编下代码，减轻看代码的工作量呢？

首先，看调用堆栈，发现所有imageMagick调用都在动态链接库libMagickCore-6.Q16.so.2中。需要down一份线上同版本的imageMagick代码（v6.9.2，实测最新代码也要同样问题），打开-g编译选项，替换掉开发环境原有的libMagickCore-6.Q16.so.2库，重复之前的调试步骤，获得新的调用堆栈。博主本地常年用的调试版so，故省略该步。。。

然后，使用objdump反汇编该动态链接库，比较大，需要耐心等待
```
objdump -l -DS ./libMagickCore-6.Q16.so.2 > image.dump
```

查找第二次OpenBlob的调用地址，.text段，偏移地址+0x30aada
```
/root/ImageMagick-6.9.1-2/coders/webp.c:250
  30aada:       85 c0                   test   %eax,%eax
  30aadc:       0f 84 92 00 00 00       je     30ab74 <ReadWEBPImage+0x124>
WebPInitDecoderConfig():
/usr/local/include/webp/decode.h:466
  30aae2:       4c 8d 6c 24 40          lea    0x40(%rsp),%r13
  30aae7:       be 03 02 00 00          mov    $0x203,%esi
  30aaec:       4c 89 ef                mov    %r13,%rdi
  30aaef:       e8 0c 1a d6 ff          callq  6c500 <WebPInitDecoderConfigInternal@plt>
```
抓到了，问题出在coders/webp.c，第250行。。。

# 5.BUG解决
定位到了出错代码位置就好办了。经过对代码的分析，不难看出，imageMagick在webp的解包器中使用OpenBlob打开了源文件，返回前忘记关闭了。。。

ps：博主发文前，提交的fix pr已被master代码合并，故上述问题官方最新代码已修复
