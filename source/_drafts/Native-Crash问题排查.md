---
title: Native-Crash问题排查
tags:
    - JNI
    - Crash
categories:
---

todo 
1. 需要加一下总结，这种不能算是技术沉淀：
我觉得问题解决的投稿要么是针对某一类问题进行解决，得出该类问题解决方法论；
要么就是教会大家解决jni问题的方法，别人可以学会你这个方案，然后去解决其他的jni问题，现在问题是解决一个具体的问题，可以再想一下你要突出什么主题

2. 最好把需要表达的主题放到文章开头，吸引大家来看；下面解决问题的过程作为一个例子来做演示


本文以一次Native Crash问题排查为例，描述该类线上问题的排查方法，主要有addr2line和objdump两种方式。

举例：http://jira.yupaopao.com/browse/CRAS-93609

```
*** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
Tombstone maker: 'xCrash 3.0.0'
Crash type: 'native'
Start time: '2021-12-27T09:30:11.898+0800'http://jira.yupaopao.com/browse/CRAS-93609
Crash time: '2021-12-27T10:33:17.592+0800'
App ID: 'com.yangle.xiaoyuzhou'
App version: '6.0.0'
Rooted: 'No'
API level: '28'
OS version: '9'
Kernel version: 'Linux version 4.9.148 #1 SMP PREEMPT Wed Nov 17 02:54:11 CST 2021 (armv8l)'
ABI list: 'arm64-v8a,armeabi-v7a,armeabi'
Manufacturer: 'HUAWEI'
Brand: 'HONOR'
Model: 'COR-AL10'
Build fingerprint: 'HONOR/COR-AL10/HWCOR:9/HUAWEICOR-AL10/9.1.0.346C00:user/release-keys'
ABI: 'arm'
pid: 16878, tid: 21754, name: Thread-76  >>> com.yangle.xiaoyuzhou <<<
signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x0
    r0  00000000  r1  00000000  r2  ffff8000  r3  00000000
    r4  00000000  r5  00007fff  r6  00000000  r7  992fc398
    r8  000003c0  r9  91360800  r10 91360000  r11 bee5ec80
    ip  992fc3c0  sp  992fc378  lr  992fc3c4  pc  9621fc18

backtrace:                                                                                                  
    #00 pc 0001fc18  /data/app/com.yangle.xiaoyuzhou-05dGXL3mMI6mFvODW6C8Rg==/lib/arm/libaudio-mix-lib.so (_ZN8AudioMix7ProcessEPsS0_S0_PS0_S1_+359)
    #01 pc 00016503  /data/app/com.yangle.xiaoyuzhou-05dGXL3mMI6mFvODW6C8Rg==/lib/arm/libaudio-mix-lib.so (Java_com_app_audio_IAudioMixer_process2+222)
    #02 pc 002e96bd  /data/app/com.yangle.xiaoyuzhou-05dGXL3mMI6mFvODW6C8Rg==/oat/arm/base.odex (com.app.audio.IAudioMixer.process [DEDUPED]+212)
    #03 pc 0003840f  /dev/ashmem/dalvik-jit-code-cache (deleted)
stack:
         93278be8  93e68000  [anon:libc_malloc]
         93278bec  91f78000  [anon:libc_malloc]
         93278bf0  91f34000  [anon:libc_malloc]
         93278bf4  9e131000  [anon:libc_malloc]
         93278bf8  9fb301c0  [anon:libc_malloc]
         93278bfc  9922e060  /data/app/com.yangle.xiaoyuzhou-2AAHLK4-zSt-1CRlIQWYwQ==/lib/arm/libaudio-mix-lib.so
         93278c00  93278c10
         93278c04  9916a68f  /data/app/com.yangle.xiaoyuzhou-2AAHLK4-zSt-1CRlIQWYwQ==/lib/arm/libaudio-mix-lib.so (_ZN11AudioRender11RenderAudioEPh+26)
         93278c08  bc285000  [anon:libc_malloc]
         93278c0c  93278cc0
         93278c10  93278c20
         93278c14  99172ca3  /data/app/com.yangle.xiaoyuzhou-2AAHLK4-zSt-1CRlIQWYwQ==/lib/arm/libaudio-mix-lib.so (_ZN8AudioMix14RenderMicAudioEPs+26)
         93278c18  abf3b9a0  [anon:libc_malloc]
         93278c1c  9922e060  /data/app/com.yangle.xiaoyuzhou-2AAHLK4-zSt-1CRlIQWYwQ==/lib/arm/libaudio-mix-lib.so
         93278c20  93278c48
         93278c24  99172b37  /data/app/com.yangle.xiaoyuzhou-2AAHLK4-zSt-1CRlIQWYwQ==/lib/arm/libaudio-mix-lib.so (_ZN8AudioMix7ProcessEPsS0_S0_PS0_S1_+134)
    #00  93278c28  93278c48
         93278c2c  93278cc4
         93278c30  91f34000  [anon:libc_malloc]
         93278c34  93278cc8
         93278c38  93e68000  [anon:libc_malloc]
         93278c3c  abf3b9a0  [anon:libc_malloc]
         93278c40  93278cc0
         93278c44  9922e060  /data/app/com.yangle.xiaoyuzhou-2AAHLK4-zSt-1CRlIQWYwQ==/lib/arm/libaudio-mix-lib.so
         93278c48  93278c98
         93278c4c  99169507  /data/app/com.yangle.xiaoyuzhou-2AAHLK4-zSt-1CRlIQWYwQ==/lib/arm/libaudio-mix-lib.so (Java_com_app_audio_IAudioMixer_process2+226)
    #01  93278c50  93278c74
```

以下是需要注意的主要信息
```
signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x0
    r0  00000000  r1  00000000  r2  ffff8000  r3  00000000
    r4  00000000  r5  00007fff  r6  00000000  r7  992fc398
    r8  000003c0  r9  91360800  r10 91360000  r11 bee5ec80
    ip  992fc3c0  sp  992fc378  lr  992fc3c4  pc  9621fc18

backtrace:
    #00 pc 0001fc18  /data/app/com.yangle.xiaoyuzhou-05dGXL3mMI6mFvODW6C8Rg==/lib/arm/libaudio-mix-lib.so (_ZN8AudioMix7ProcessEPsS0_S0_PS0_S1_+359)
    #01 pc 00016503  /data/app/com.yangle.xiaoyuzhou-05dGXL3mMI6mFvODW6C8Rg==/lib/arm/libaudio-mix-lib.so (Java_com_app_audio_IAudioMixer_process2+222)
    #02 pc 002e96bd  /data/app/com.yangle.xiaoyuzhou-05dGXL3mMI6mFvODW6C8Rg==/oat/arm/base.odex (com.app.audio.IAudioMixer.process [DEDUPED]+212)
    #03 pc 0003840f  /dev/ashmem/dalvik-jit-code-cache (deleted)
```
根据Linux Signal，可以了解到信号量-11 对应 SIGSEGV，错误代码-1 对应 SEGV_MAPERR，报错的内存地址为 0x0。以下的从r0、r1到pc是寄存器信息，暂时不用太过关注。

SIGSEGV：SIG是信号名的通用前缀，SEGV是段违法的缩写，SIGSEGV一般发生内存操作时，比如__memcpy_base、memcpy等
SEGV_MAPERR：表示堆栈映射错误

以上基本得出某个参数指针的内存地址为空，发生了空指针异常。

下面的回溯信息就比较关键了，从左到右依次是栈帧，所处寄存器，地址信息，问题路径，主要关注点在于'0001fc18'和'00016503'这两个地址信息，可以利用add2line工具和objdump工具来定位到对应so的代码方法。
注意这里使用到的so是对应平台arm的release编译(版本0.3.57)，且未去symbol，否则无法获得对应问题代码！

### add2line

```
dzsb-002300@clouds-MacBook-Pro bin % aarch64-linux-android-addr2line -e ~/Downloads/test/release/obj/armeabi-v7a/libaudio-mix-lib.so 0001fc18 00016503
/Users/dzsb-002300/ypp/ypp-audio-android/audio/src/main/cpp/AudioMix/AudioMix.cpp:246
/Users/dzsb-002300/Library/Android/sdk/ndk/21.0.6113669/toolchains/llvm/prebuilt/darwin-x86_64/sysroot/usr/include/jni.h:905
```
p.s. addr2line工具路径在 '/Users/dzsb-002300/Library/Android/sdk/ndk/21.0.6113669/toolchains/aarch64-linux-android-4.9/prebuilt/darwin-x86_64/bin/' 文件夹下，'-e'参数是为了指定需要转换地址的可执行文件名，这里指定的so文件是通过audio-mix库源码编译的release库

根据Terminal输出可以看到问题代码是在 'AudioMix.cpp' line.246

上述backtrace日志比较完整，可以清晰地得到add2line的地址（0001fc18、00016503），实际上日志文件可能会不完整，比如只拿到了stack内容，这时候可以根据stack内的函数调用情况反推出相应的函数地址，再用addr2line进行转换。

这里要先说明基地址的概念，Library Base Address（共享库在内存中基地址）。JNI在运行时可以看到在java中有'loadLibrary'的调用，可以理解为将一份so库文件加载到内存中。因此这份so库在内存中就存在一个加载的基地址，但是根据内存的情况，该地址每次都可能不一样。而addr2line需要的地址其实是相对于共享库的一个绝对地址（排除共享库加载地址的影响），所以如果得到了共享库在内存中的基地址，就可以通过stack上的地址计算出可用的add2line地址。
在上面的stack和backtrace信息中可以得到(Java_com_app_audio_IAudioMixer_process2+226)和(Java_com_app_audio_IAudioMixer_process2+222)这两个symbol的相对地址和绝对地址。
0x99169507-0x16503-0x4=0x99153000
得到基地址为0x99153000，通过该地址可以验证(_ZN8AudioMix7ProcessEPsS0_S0_PS0_S1_+134)和(_ZN8AudioMix7ProcessEPsS0_S0_PS0_S1_+359)这两个symbol的地址
0x99172b37-0x99153000+0xe1=0x1fc18

大多数情况下是不需要通过基地址来计算可用的addr2line地址的，这里只是为了大家了解到共享库加载的相关知识。

### objdump

```
// objdump工具路径和addr2line一致，为方便使用可以在环境变量配置alias
dzsb-002300@clouds-MacBook-Pro bin % aarch64-linux-android-objdump -d ~/Downloads/test/audio/release/obj/armeabi-v7a/libaudio-mix-lib.so > ~/Downloads/test/audio/dump.log
```

得到dump.log文件，文件内搜索'1fc18'，可以进一步把问题确认在RenderMicAudio函数中

这里根据tombstone显示的地址和偏移量无法对应'1fc18'（十六进制的'1fa38'加上十进制的'359'无法对齐十六进制的'1fc18'）也是由于上述的基地址偏移导致的，准确的计算可以通过stack中的(_ZN8AudioMix7ProcessEPsS0_S0_PS0_S1_+134)
查找'<_ZN8AudioMix7ProcessEPsS0_S0_PS0_S1_>'，0x1fa38+0x86=0x1fabe，这个'0x1fabe'和'0x1fc18'有一定的差别，原因是stack上保存的是函数的返回地址，但是指向的指令是相同的。
```
0001fa38 <_ZN8AudioMix7ProcessEPsS0_S0_PS0_S1_>:
1fa38:	b5f0      	push	{r4, r5, r6, r7, lr}
   1fa3a:	af03      	add	r7, sp, #12
   1fa3c:	e92d 0f80 	stmdb	sp!, {r7, r8, r9, sl, fp}
   1fa40:	4699      	mov	r9, r3
   1fa42:	4692      	mov	sl, r2
   1fa44:	4683      	mov	fp, r0
   1fa46:	b169      	cbz	r1, 1fa64 <_ZN8AudioMix7ProcessEPsS0_S0_PS0_S1_+0x2c>
   1fa48:	f8db 0000 	ldr.w	r0, [fp]
   1fa4c:	4688      	mov	r8, r1
   1fa4e:	2802      	cmp	r0, #2
   // ....
   1faa0:	4649      	mov	r1, r9
   1faa2:	f7f5 ee88 	blx	157b4 <_ZN8AudioMix21SingleChannelToDoubleEPsS0_@plt>
   1faa6:	f8db 9034 	ldr.w	r9, [fp, #52]	; 0x34
   1faaa:	e001      	b.n	1fab0 <_ZN8AudioMix7ProcessEPsS0_S0_PS0_S1_+0x78>
   1faac:	f04f 0900 	mov.w	r9, #0
   1fab0:	f1b8 0f00 	cmp.w	r8, #0
   1fab4:	d004      	beq.n	1fac0 <_ZN8AudioMix7ProcessEPsS0_S0_PS0_S1_+0x88>
   1fab6:	4658      	mov	r0, fp
   1fab8:	4641      	mov	r1, r8
   1faba:	f7f5 ee82 	blx	157c0 <_ZN8AudioMix14RenderMicAudioEPs@plt>
   1fabe:	e000      	b.n	1fac2 <_ZN8AudioMix7ProcessEPsS0_S0_PS0_S1_+0x8a>
   1fac0:	2000      	movs	r0, #0
   1fac2:	f8db 101c 	ldr.w	r1, [fp, #28]
   1fac6:	f647 75ff 	movw	r5, #32767	; 0x7fff
   1faca:	e9d7 ec02 	ldrd	lr, ip, [r7, #8]

0001fc10 <_ZN8AudioMix14RenderMicAudioEPs>:
   1fc10:	b5d0      	push	{r4, r6, r7, lr}
   1fc12:	af02      	add	r7, sp, #8
   1fc14:	69c2      	ldr	r2, [r0, #28]
   1fc16:	4604      	mov	r4, r0
   1fc18:	6b80      	ldr	r0, [r0, #56]	; 0x38
   1fc1a:	0092      	lsls	r2, r2, #2
   1fc1c:	f7f4 eee0 	blx	149e0 <__aeabi_memcpy@plt>
   1fc20:	6be0      	ldr	r0, [r4, #60]	; 0x3c
   1fc22:	b110      	cbz	r0, 1fc2a <_ZN8AudioMix14RenderMicAudioEPs+0x1a>
   1fc24:	6ba1      	ldr	r1, [r4, #56]	; 0x38
   1fc26:	f7f4 ef48 	blx	14ab8 <_ZN11AudioRender11RenderAudioEPh@plt>
   1fc2a:	6ba0      	ldr	r0, [r4, #56]	; 0x38
   1fc2c:	bdd0      	pop	{r4, r6, r7, pc}
```

### 结论

通过add2line工具和objdump工具联合定位

```
79 void AudioMix::Process(short* micBuffer, short* bgmBuffer, short* accomBuffer, short** mixOutBuffer, short** mixBgmAccomOutBuffer)
80 {
81	short* micBuf = micBuffer;
    ....
99	if (nullptr != micBuf)
100	{
101		micBuf = RenderMicAudio(micBuf);
102	}
    ....
121 }

244 short* AudioMix::RenderMicAudio(short* micBuffer)
245 {
246 	memcpy(m_micRenderBuffer, micBuffer, 2 * m_samples * sizeof(short));
247 	//render
248 	if (nullptr != m_render)
249 	{
250 		m_render->RenderAudio((unsigned char*)m_micRenderBuffer);
251 	}
252 	return m_micRenderBuffer;
253 }
```

可以得出crash发生在'memcpy'方法中，但是明明在调用前判断了micBuffer是否为空，这时把怀疑的重点投向了'm_micRenderBuffer'

```
//audio/src/main/cpp/AudioMix/AudioMix.cpp
AudioMix::AudioMix(int micChannel, int bgmChannel, int accomChannel, int sampleRate, int samples, int bytesPerSample)
: m_micChannel(micChannel)
, m_bgmChannel(bgmChannel)
, m_accomChannel(accomChannel)
, m_sampleRate(sampleRate)
, m_samples(samples)
, m_bytesPerSample(bytesPerSample)
{
	.....
	m_micRenderBuffer = new short[2 * samples];
	....
}

AudioMix::~AudioMix()
{
	....
	delete[] m_micBuffer;
	m_micBuffer = nullptr;
	....
}
```
```
//audio/src/main/cpp/IAudioMix.cpp
extern "C"
JNIEXPORT void JNICALL
Java_com_app_audio_IAudioMixer_init(JNIEnv *env, jclass type,
                                    jint micChannel, jint bgmChannel, jint accomChannel,
                                    jint sampleRate, jint samples, jint bytesPerSample) {

    LOGD("Java_com_app_audio_IAudioMixer_init +");
    if (s_pAudioMix) {
        return;
    }
    s_pAudioMix = new AudioMix(micChannel, bgmChannel, accomChannel, sampleRate, samples,
                               bytesPerSample);
    s_out_buf_size = 2 * samples * sizeof(short);
    LOGD("Java_com_app_audio_IAudioMixer_init -");
}

extern "C"
JNIEXPORT void JNICALL
Java_com_app_audio_IAudioMixer_unInit(JNIEnv *env, jclass type) {

    LOGD("Java_com_app_audio_IAudioMixer_unInit +");
    unInit();
    LOGD("Java_com_app_audio_IAudioMixer_unInit -");
}

static inline void unInit() {
    if (s_pAudioMix) {
        delete s_pAudioMix;
        s_pAudioMix = nullptr;
    }
}
```
可以看到 'm_micRenderBuffer' 的初始化和释放分别发生在类加载和析构方法中，分别对应java方法中的init()和uninit()，由业务方控制。

这时就需要排查业务方代码，调用如下：
```java
    private fun initMix() {
        IAudioMixer.init(2, 2, 2, 48000, 480, 2)
        IAudioMixer.enableAudioEffect(true)
        IAudioMixer.enableSoundTouch(true)
    }

    private fun unInitMix() {
        IAudioMixer.enableAudioEffect(false)
        IAudioMixer.enableSoundTouch(false)
        IAudioMixer.unInit()
    }

    private fun initZegoCapture() {
        ....
        mZegoLiveRoom?.initSDK(SecurityService.getInstance().key5, SecurityService.getInstance().key6) { i: Int ->
            LogUtil.i("[SonaLive][TRTCMultiAudioLink] ZEGO initSDK:$i")
            ....
            mZegoLiveRoom?.setAudioPrepCallback({ inFrame ->
                val outFrame = ZegoAudioFrame()
                outFrame.frameType = inFrame.frameType
                outFrame.samples = inFrame.samples
                outFrame.bytesPerSample = inFrame.bytesPerSample
                outFrame.channels = inFrame.channels
                outFrame.sampleRate = inFrame.sampleRate
                outFrame.timeStamp = inFrame.timeStamp
                outFrame.configLen = inFrame.configLen
                outFrame.bufLen = inFrame.bufLen

                val audioFrame = TRTCAudioFrame()
                val byteArray = ByteArray(inFrame.bufLen)
                inFrame.buffer.get(byteArray, 0, inFrame.bufLen)
                audioFrame.data = byteArray
                audioFrame.sampleRate = 48000
                audioFrame.channel = 2
                if (voiceType != VoiceType.ORIGINAL) {
                    IAudioMixer.setSoundTouchDouble(voiceType.type)
                    val size: Int = inFrame.bufLen
                    val bgmBuffer = ByteArray(size)
                    val accomBuffer = ByteArray(size)
                    val mixOutBuffer = ByteArray(size)
                    val mixBgmAccomOutBuffer = ByteArray(size)
                    IAudioMixer.process2(byteArray, bgmBuffer, accomBuffer, mixOutBuffer, mixBgmAccomOutBuffer)
                    System.arraycopy(mixOutBuffer, 0, byteArray, 0, size)
                    outFrame.buffer = byte2Buffer(byteArray)
                } else {
                    outFrame.buffer = inFrame.buffer.duplicate()
                }
                rtc?.sendCustomAudioData(audioFrame)
                outFrame
            }, config)
            mZegoLiveRoom?.startPreview()
        }
    }
```

可以看到 IAudioMixer 的 setSoundTouchDouble() 方法、process2()方法调用与 init/uninit 状态无关，在zego的callback方法回调中就可能导致 IAudioMixer 已经析构，却仍然调用了process2()方法，此时就会报出上述的JNI Crash。

改动：根据 IAudioMixer 的 init 与 uninit 状态设置变量，根据是否初始化来判断调用 setSoundTouchDouble() 与 process2()

总结：每个JNI的问题定位都需要结合业务代码一起判断

参考链接：

https://www.twblogs.net/a/5b7ce90b2b71770a43dd1d34
