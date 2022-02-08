---
title: booster源码解析
tags:
categories:
---

包体治理

图片压缩

booster-task-compression-cwebp

booster-task-compression-pngquant

booster-pngquant-provider

zip文件压缩

booster-task-compression-processed-res

资源索引内联

booster-transform-r-inline

移除冗余资源

booster-task-resource-deredundancy

接下来主要介绍的是图片压缩

分析booster-task-compression-cwebp、booster-task-compression-pngquant前，要聊一下booster-task-compression，它提供了图片处理的接口类，

booster-pngquant-provider 内置了 linux/macos/windows 三端的 pngquant 可执行文件，pngquant 是有损压缩，pngguant为什么单独独立出来，是因为 license 的原因，这个