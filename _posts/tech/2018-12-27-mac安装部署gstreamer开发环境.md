---
layout: post
title: 如何在Mac上安装部署gstreamer开发环境
category: 技术
tags: gstreamer
comments: true
---

# 如何在Mac上安装部署gstreamer开发环境
> 当前开发环境：
> macOS Sierra 10.12.4
> gstreamer 1.14.4

## 下载gstreamer安装包
[下载路径](https://gstreamer.freedesktop.org/data/pkg/osx/)

运行时库 https://gstreamer.freedesktop.org/data/pkg/osx/1.14.4/gstreamer-1.0-1.14.4-x86_64.pkg

开发包 https://gstreamer.freedesktop.org/data/pkg/osx/1.14.4/gstreamer-1.0-devel-1.14.4-x86_64.pkg

## 添加pkg-config路径
添加如下环境变量

```
export PKG_CONFIG_PATH=/Library/Frameworks/GStreamer.framework/Versions/Current/lib/pkgconfig
```

## 编译环境配置
使用xcode请按照官方文档的指导部署[链接](https://gstreamer.freedesktop.org/documentation/installing/on-mac-osx.html#configure-your-development-environment)

目前使用的是gcc编译，使用了上述pkg-config的配置后，可以使用跟linux一样的编译连链接命令
```
gcc basic-tutorial-9.c -o basic-tutorial-9 `pkg-config --cflags --libs gstreamer-1.0 gstreamer-pbutils-1.0`
```

## OSX特定elements
* [osxvideosink](https://gstreamer.freedesktop.org/documentation/tutorials/basic/platform-specific-elements.html#osxvideosink)
* [osxaudiosink](https://gstreamer.freedesktop.org/documentation/tutorials/basic/platform-specific-elements.html#osxaudiosink)
