---
title: So安全治理
tags:
    - JNI
    - NDK
categories:
---

cmake 

makefile

静态注册改为动态注册

JNI_ONLOAD

JNINativeMethod

编译参数添加 

cmake --> add_compile_options(-fPIC)
makefile --> LOCAL_CFLAGS := - fPIC
