---
title: IjkPlayer疑难杂症
tags:
categories:
---

1.h265硬解设置失败

```java
player.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "mediacodec", 1);
// 改为
player.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "mediacodec-all-videos", 1);
```

参考资料：

https://juejin.im/post/5bc7e689f265da0adc18fb7a  
