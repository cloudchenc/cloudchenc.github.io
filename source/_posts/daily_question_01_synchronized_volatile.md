---
title: 「每日一问」聊聊 synchronized, volatile
date: 2020-08-06 19:17:52
tags:
    - 并发编程
categories: 每日一问
---

synchronized

```java 
class X {
  // 修饰非静态方法
  synchronized void foo() {
    // 临界区
  }
  // 修饰静态方法
  synchronized static void bar() {
    // 临界区
  }
  // 修饰代码块
  Object obj = new Object()；
  void baz() {
    synchronized(obj) {
      // 临界区
    }
  }
}  
```

当修饰静态方法的时候，锁定的是当前类的 Class 对象，在上面的例子中就是 Class X；当修饰非静态方法的时候，锁定的是当前实例对象 this。


### synchronized 与 volatile 的异同



### 多线程下如何解决线程安全问题

### synchronized：类锁和对象锁的区别

