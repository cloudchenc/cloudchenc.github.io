---
title: 记录一次JNI Crash排查
tags:
    - JNI
    - Crash
categories:
---

偶现的JNI Crash问题排查

迫于名下的JIRA问题，展开了一次JNI Crash排查

http://jira.yupaopao.com/browse/CRAS-93609

```
*** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
Tombstone maker: 'xCrash 3.0.0'
Crash type: 'native'
Start time: '2021-12-27T09:30:11.898+0800'
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
注意这里使用到的so是对应平台arm的release编译(版本0.3.57)，且未strip，否则无法获得对应问题代码！

### add2line

```
dzsb-002300@clouds-MacBook-Pro bin % aarch64-linux-android-addr2line -e ~/Downloads/test/release/obj/armeabi-v7a/libaudio-mix-lib.so 0001fc18 00016503
/Users/dzsb-002300/ypp/ypp-audio-android/audio/src/main/cpp/AudioMix/AudioMix.cpp:246
/Users/dzsb-002300/Library/Android/sdk/ndk/21.0.6113669/toolchains/llvm/prebuilt/darwin-x86_64/sysroot/usr/include/jni.h:905
```
p.s. addr2line工具路径在 '/Users/dzsb-002300/Library/Android/sdk/ndk/21.0.6113669/toolchains/aarch64-linux-android-4.9/prebuilt/darwin-x86_64/bin/' 文件夹下，'-e'参数是为了指定需要转换地址的可执行文件名，这里指定的so文件是通过audio-mix库源码编译的release库

根据Terminal输出可以看到问题代码是在 'AudioMix.cpp' line.246

### objdump

```
// objdump工具路径和addr2line一致，为方便使用可以在环境变量配置alias
dzsb-002300@clouds-MacBook-Pro bin % aarch64-linux-android-objdump -d ~/Downloads/test/audio/release/obj/armeabi-v7a/libaudio-mix-lib.so > ~/Downloads/test/audio/dump.log
```

得到dump.log文件，文件内搜索'1fc18'，可以进一步把问题确认在RenderMicAudio函数中

唯一困扰我的是根据tombstone显示的地址和偏移量无法对应'1fc18'（十六进制的'1fa38'加上十进制的'359'无法对齐十六进制的'1fc18'）
```
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
