---
title: sona
tags:
categories:
---

## sona

+ sona
+ sonabase
+ sonademo
+ sonagame
+ sonareport
+ sonaauido
+ sonasonacompiler

### sonareport

日志上报，打印，写入文件

代理模式，封装，继承，多态

> IReport：上报接口，封装上报和过滤。  
>
> ReporterDelegate：代理类，创建HandlerThread，负责实际调用 IReport 的实现类（LogReport, LoganReport, ApiReport, DemeterReport, SorakaReport）report()方法  
>
> SorakaReport: Soraka日志上报，走的是soraka library的接口  
> LoganReport: 将SonaRoom日志通过Logan写入  
> DemeterReport: 记录Demeter日志事件  
> LogReport: 打印普通日志  
> ApiReport: 通过接口上报日志  

### sonagame

> GameComponentBridge：游戏组件内部通信封装，实现 sona消息和游戏内部消息 分发，注册/注销 监听等  
> GameComponent：实现了游戏组件，启动游戏，渲染游戏，离开游戏，以及收到游戏内部消息的回调，设置游戏渲染器  
> GameLifeCycle：监听生命周期，初始化/销毁 游戏配置  
>
> Render: 接口类，定义方法，设置游戏渲染器，开始渲染，停止渲染，离开游戏  
> RenderImpl：渲染器的具体实现，startRender添加surface，调用  
>> startRender：添加 surface，调用 render 进行渲染，同时通过 GameComponentBridge 发送 GAME_RENDER_START 消息  
>> stopRender：移除 surface，调用 render 停止渲染，通过 GameComponentBridge 发送 GAME_RENDER_END 消息  
>
> GameRenderFactory：实现 createRender，根据游戏类型初始化渲染器，NativeRender or WebGameRender  
> NativeRender: 实现 executeLeaveGame 调用接口离开游戏  
> WebGameRender：实现 convertRootView 创建 layout 作为 h5 载体，实现 executeLeaveGame 调用接口离开游戏  

### sonademo

sona unit code

### sonacompiler

JavaPoet 自动生成代码，目的是为了解耦

### sonabase

annotation 注解使用

### sonaaudio

> AudioReportCode: 即构、腾讯SDK错误码  
> CatStrategy：监控策略，开始、结束、执行  
> 
> IAudioComponentProvider：接口类，提供音频组件包装类  
> IStream：接口类，封装推拉流、静音流，及音频类型  
> IStreamDelegate：接口类，封装推拉流、静音流  
> IStreamFinder：接口类，查找音频信息，获取当前音频信息  
> 
> IStreamProvider：接口类，提供流  
> IStreamAcquire：接口类，获取流  
> StreamAdapter：实现 IStreamProvider, IStreamAcquire，根据不同的 流类型 返回 AudioStream
>
> AudioComponent：音频组件封装方法
> AudioComponentWrapper：音频组件包装类，添加来电监听，音频初始化
>
> 音频处理：  
> AudioVoiceMixer：开启变声
> AudioNoiseSuppressor：降噪
> 

#### 自建

> BXCatStrategy: 自建监控策略
>
> MixStream: 

#### 即构

> ZegoCatStrategy：即构监控策略
>
> MixStream: 
>
> MultiStream: 
>
> ZegoPlayer: 即构背景音乐播放器
>
> ZegoAudio: 
>
> ZegoStream: 

#### 腾讯

> MixStream: 
>
> MultiStream: 
>
> TencentPlayer: 
>
> TencentAudio: 
>
> TencentStream: 

### sona





