---
title: WebView性能优化
tags:
categories:
---

#### 1. 前端优化的局限
+ 无法规避 WebView 初始化耗时
+ 受限于 WebView 生命周期范围

终端初始化
--> Web页面主请求
--> 页面解析，资源加载和处理
--> 内容请求和处理
--> 加载完成

参考：

https://juejin.im/post/5f10446af265da230d324813