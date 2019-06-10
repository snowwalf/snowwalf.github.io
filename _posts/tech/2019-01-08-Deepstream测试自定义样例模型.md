---
layout: post
title: Deepstream测试自定义样例模型
category: 技术
tags: deepstream
comments: true
---

# 1.基础依赖部署


## 1.1.安装tensorflow
* `sudo pip install tensorflow --ignore-installed`

## 1.2.安装tensorrt
* 下载tensorrt的开发包
* `sudo pip2 install ./python/tensorrt-5.0.2.6-py2.py3-none-any.whl`
* `sudo pip2 install ./uff/uff-0.5.5-py2.py3-none-any.whl`
* `sudo pip2 install ./graphsurgeon/graphsurgeon-0.3.2-py2.py3-none-any.whl`
* `export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:<PATH_TO_TENSORRT>/lib`

# 2.测试objectDetector_SSD

## 2.1.下载ssd模型包
* `wget http://download.tensorflow.org/models/object_detection/ssd_inception_v2_coco_2017_11_17.tar.gz`
* `tar -zxvf ssd_inception_v2_coco_2017_11_17.tar.gz`

## 2.2.转uff模型

* `cp <PATH_TO_TENSORRT>/samples/sampleUffSSD/config.py .`
* `python2 /usr/local/lib/python2.7/dist-packages/uff/bin/convert_to_uff.py --input-file frozen_inference_graph.pb -O NMS -p config.py`
* `cp frozen_inference_graph.uff <PATH_TO_DEEPSTREAM>/sources/objectDetector_SSD/sample_ssd_relu6.uff`
* `cp <PATH_TO_TENSORRT>/data/ssd/ssd_coco_labels.txt <PATH_TO_DEEPSTREAM>/sources/objectDetector_SSD/ssd_coco_labels.txt`

## 2.3.测试指令
```
gst-launch-1.0 filesrc location=../../samples/streams/sample_720p.mp4 ! decodebin ! nvinfer config-file-path= config_infer_primary_ssd.txt ! nvvidconv ! nvosd ! nveglglessink
```

# 3.测试FasterRCNN

## 3.1.下载faster_rcnn模型包
* `wget https://dl.dropboxusercontent.com/s/o6ii098bu51d139/faster_rcnn_models.tgz`
* `tar -zxvf faster_rcnn_models.tgz`

## 3.2.部署模型
* `cp <PATH_TO_TENSORRT>/data/faster-rcnn/faster_rcnn_test_iplugin.prototxt .`
* `cp faster_rcnn_models/VGG16_faster_rcnn_final.caffemodel .`

## 3.3.测试指令
```
gst-launch-1.0 filesrc location=../../samples/streams/sample_720p.mp4 ! decodebin ! nvinfer config-file-path= config_infer_primary_fasterRCNN.txt ! nvvidconv ! nvosd ! nveglglessink
```