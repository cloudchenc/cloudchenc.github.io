---
title: Android开发艺术探索-第三章
tags:
categories:
---

1. 

2. 假设事件交由子元素处理，如果父容器在ACTION_UP时返回了true，就会导致子元素无法接收到ACTION_UP事件，这个时候子元素中的onClick事件就无法触发，但是父容器比较特殊，一旦它开始拦截任何一个事件，那么后续的事件都会交给它来处理，而ACTION_UP作为最后一个事件也必定可以传递给父容器，即便父容器的onInterceptTouchEvent方法在ACTION_UP时返回了false。
