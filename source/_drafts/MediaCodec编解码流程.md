---
title: MediaCodec 编解码流程
tags:
    - MediaCodec
categories: 音视频
---

Android 4.1 提供了 MediaCodec 接口来访问设备的编解码器，不同于 FFmpeg 的软件编解码，它采用的是硬件编解码，速度上会更有优势。

但是由于Android碎片化严重，机型众多，需要 MediaCodec 做一定的适配，并且编解码流程全部依赖厂商的底层硬件，最终得到的视频质量并不可靠。

相机预览的 YUV 数据编码为 H264 视频举例 MediaCodec 的使用

#### 工作模型

典型的生产者消费者模型，输入端是生产者，输出端是消费者。

输入端将数据交给 MediaCodec 进行编码或者解码，而输出端就得到编码或者解码后的内容。

输入端和输出端是通过输入队列缓冲区和输出队列缓冲区，两条缓冲区队列的形式来和 MediaCodec 传递数据。

首先从输入队列中出队得到一个可用的缓冲区，将它填满数据之后，再将缓冲区入队，交由 MediaCodec 去处理。

MediaCodec 处理完了之后，再从输出队列中出队得到一个可用的缓冲区，这个缓冲里面的数据就是编码或者解码后的数据了，把这些数据进行相应的处理之后，还需要释放这个缓冲区，让它回到队列中去，可供下一次使用。

#### 生命周期

##### Uninitialized

```java
MediaCodec codec = new MediaCodec();
```

##### Configured

`MediaCodec#configure(MediaFormat format, Surface surface, MediaCrypto crypto, int flags)`

###### MediaFormat

+ 采样率
+ 分辨率
+ 颜色格式
+ 码率
+ 帧率
+ I帧间隔

##### Executing

`MediaCodec#start()`

###### Flushed

`MediaCodec#flush()`

###### Running

###### End of Stream

#### 调用流程

```java
// 创建 MediaCodec，此时是 Uninitialized 状态
 MediaCodec codec = MediaCodec.createByCodecName(name);
 // 调用 configure 进入 Configured 状态
 codec.configure(format, …);
 MediaFormat outputFormat = codec.getOutputFormat(); // option B
 // 调用 start 进入 Executing 状态，开始编解码工作
 codec.start();
 for (;;) {
   // 从输入缓冲区队列中取出可用缓冲区，并填充数据
   int inputBufferId = codec.dequeueInputBuffer(timeoutUs);
   if (inputBufferId >= 0) {
     ByteBuffer inputBuffer = codec.getInputBuffer(…);
     // fill inputBuffer with valid data
     …
     codec.queueInputBuffer(inputBufferId, …);
   }
   // 从输出缓冲区队列中拿到编解码后的内容，进行相应操作后释放，供下一次使用
   int outputBufferId = codec.dequeueOutputBuffer(…);
   if (outputBufferId >= 0) {
     ByteBuffer outputBuffer = codec.getOutputBuffer(outputBufferId);
     MediaFormat bufferFormat = codec.getOutputFormat(outputBufferId); // option A
     // bufferFormat is identical to outputFormat
     // outputBuffer is ready to be processed or rendered.
     …
     codec.releaseOutputBuffer(outputBufferId, …);
   } elseif (outputBufferId == MediaCodec.INFO_OUTPUT_FORMAT_CHANGED) {
     // Subsequent data will conform to new format.
     // Can ignore if using getOutputFormat(outputBufferId)
     outputFormat = codec.getOutputFormat(); // option B
   }
 }
 // 调用 stop 方法进入 Uninitialized 状态
 codec.stop();
 // 调用 release 方法释放，结束操作
 codec.release();
```



参考：  
https://mp.weixin.qq.com/s/RULqfsd0VMci1JLy0V3bKw  
https://www.polarxiong.com/archives/Android-MediaCodec%E8%A7%86%E9%A2%91%E6%96%87%E4%BB%B6%E7%A1%AC%E4%BB%B6%E8%A7%A3%E7%A0%81-%E9%AB%98%E6%95%88%E7%8E%87%E5%BE%97%E5%88%B0YUV%E6%A0%BC%E5%BC%8F%E5%B8%A7-%E5%BF%AB%E9%80%9F%E4%BF%9D%E5%AD%98JPEG%E5%9B%BE%E7%89%87-%E4%B8%8D%E4%BD%BF%E7%94%A8OpenGL.html

