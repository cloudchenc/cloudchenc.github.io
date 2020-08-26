---
title: PagerAdapter
tags:
categories: Android
---

### 使用场景

ViewPager + PagerAdapter

PagerAdapter 是提供填充 ViewPager 内部页面的适配器，而 ViewPager 负责控制每一个界面构造和销毁的时机，使用回调来通知 PagerAdapter 

### 抽象方法

#### #instantiateItem(ViewGroup, int)

#### #destroyItem(ViewGroup, int, Object)

#### #getCount()

#### #isViewFromObject(View, Object)

### 子类

#### FragmentPagerAdapter

#### FragmentStatePagerAdapter

### 总结
