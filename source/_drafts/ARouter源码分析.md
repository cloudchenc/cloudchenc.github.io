---
title: ARouter源码分析
tags:
categories:
---

使用APT技术

分组 + 按需加载 解决一次性加载全部映射关系的问题

分组：WareHouse + IRouteGroup + RouteMeta

```
// WareHouse
static Map<String, Class<? extends IRouteGroup>> groupsIndex = new HashMap<>();
static Map<String, RouteMeta> routes = new HashMap<>();
```

+ groupsIndex 保存了group名字到IRouteGroup类的映射，这一层映射就是Arouter新增的一层映射关系。
+ routes 保存了路径path到RouteMeta的映射，其中，RouteMeta是目标地址的包装。这一层映射关系跟我门自己方案里的map是一致的，我们路径跳转也是要用这一层来映射。


https://www.cnblogs.com/jymblog/p/11698914.html