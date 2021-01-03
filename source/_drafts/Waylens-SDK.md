---
title: Waylens SDK
tags:
categories:
---

### VideoSDK



### DeviceSDK

#### 设备接口

##### 连接机制，状态管理

已完成

##### 通信协议封装、解析

已完成

##### 相机接口封装

========================= 统一接口 =========================

- getName(): 获取相机名称
- setName(String name): 设置相机名称

- getApiVersion(): 获取相机版本号
- getBspFirmware(): 获取相机固件版本号

- getPassword(): 获取相机密码
- getCameraServer(): 获取相机连接服务器地址
- setCameraServer(String server): 设置相机连接服务器地址
- prepareLog(): 获取相机日志
- queryStorageState(): 获取相机存储状态
- getMonitorMode(): 获取相机录制状态
- factoryReset(): 恢复相机出厂值
- setMicEnabled(boolean enable): 设置相机麦克风开启or关闭

- setAudioPromptsEnabled(boolean enable): 设置相机提示音开启or关闭
- isPromptsEnabled(): 获取相机提示音开关

- sendNewFirmware(int size, String md5, Listener listener): 向相机发送固件

- getMountSettings(): 获取相机设置信息
- setMountSettings(String setting): 修改相机设置

- getHdrMode(): 获取相机HDR状态
- setHdrMode(int mode): 设置相机HDR状态

- sendFormatSDCard(): 格式化相机SD卡

- markLiveVideo(): 标记相机当前预览

- stopRecording(): 停止相机录制
- startRecording(): 开始相机录制

- getRadarSensitivity(): 获取相机雷达灵敏度
- setRadarSensitivity(int sense): 设置相机雷达灵敏度

- getMountAccTrust(): 获取相机Acc是否信任
- setMountAccTrust(boolean trust): 设置相机Acc是否信任

- getP2pEnable(): 获取相机是否开启Wifi-Direct
- setP2pEnable(boolean enable): 设置相机是否开启Wifi-Direct

- getPairedDevices(): 获取相机已配对的设备
- removePaired(String mac): 移除相机Wifi-Direct配对设备

- getIsLensNormal(): 获取相机正装or倒装
- setLensNormal(boolean isNormal): 设置相机正装or倒装

- getApnSetting(): 获取相机APN设置
- setApnSetting(String apn): 设置相机APN参数

- getSupportWlan(): 获取相机是否支持WLan

- getSupportRiskEvent(): 获取相机是否支持危险事件检测
- setSupportRiskEvent(boolean support): 设置相机是否支持危险事件检测

- getEventParam(): 获取相机事件检测参数
- setEventParam(String param): 设置相机事件检测参数

- getProtectVoltage(): 获取相机保护电压
- setProtectVoltage(int vol): 设置相机保护电压

- getParkSleepDelay(): 获取相机停车模式下延迟休眠时间
- setParkSleepDelay(int delay): 设置相机停车模式下延迟休眠时间

- getModemVersion(): 获取相机LTE固件版本

- getMarkStorage(): 获取相机事件视频存储大小
- setMarkStorage(int index): 设置相机事件视频存储大小

- getMountLevel(): 获取相机事件检测加速度等级
- setMountLevel(int index): 设置相机事件检测加速度等级

- getWifiMode(): 获取相机Wifi模式

- getIccid(): 获取iccid

- getDeviceTime(): 获取相机设备时间
- setDeviceTime(long syncTime, long syncTimeZone): 设置相机设备时间

- getMountSensitivity(): 获取相机事件监测灵敏度

- isNightVisionInDrivingAvailable(): 判断相机是否支持开车模式下的夜视
- isPowerCordTestAvailable(): 判断相机是否支持电源线检测
- isMarkSpaceSettingsAvailable(): 判断相机是否支持事件视频存储大小设置

- isAudioPromptsAvailable(): 判断相机是否支持提示音
- isModemVersionAvailable(): 判断相机是否支持获取LTE固件版本
- isNetworkTestDiagnosisAvailable(): 判断相机是否支持网络检测

- isHDRAutoAvailable(): 判断相机是否支持自动开启HDR
- isNightVisionAutoAvailable(): 判断相机是否支持自动开启夜视
- isSupportUntrustACCWireAvailable(): 判断相机是否支持不信任Acc

- isWifiDirectAvailable(): 判断相机是否支持Wifi-Direct
- isDrivingModeTimeoutSettingsAvailable(): 判断相机是否支持开车模式下的超时时间设置
- isProtectionVoltageAvailable(): 判断相机是否支持低电压保护设置

- isAPNSettingsAvailable(): 判断相机是否支持APN设置
- isSubStreamOnlyAvailable(): 判断相机是否支持单路录制
- isWLanModeAvailable(): 判断相机是否支持WLan模式

- isRiskyDrivingEventAvailable(): 判断相机是否支持危险驾驶事件检测
- isVinMirrorAvailable(): 判断相机是否支持镜头参数配置
- isRecordConfigAvailable(): 判断相机是否支持录制参数配置

- isCalibCameraAvailable(): 判断相机是否支持Calibration

========================= 统一接口 =========================


========================= TW02 =========================

- getGpsState(): 获取相机GPS状态
- refreshBatteryInfo(): 获取相机电源状态

- setWifiMode(int mode): 设置相机Wifi模式
- getHostList(): 获取相机host地址列表

- setNetworkRmvHost(String ssid): 移除相机host地址
- addNetworkHost(String ssid, String pwd): 添加相机host地址
- connectNetworkHost(String ssid): 连接相机host地址

- setStreamQuality(): 设置相机录制视频质量
- getMainStreamQuality(): 获取相机主流视频质量
- getSubStreamQuality(): 获取相机子流视频质量

- getStreamSetting(): 获取相机是否为单路录制
- setStreamSetting(boolean only): 设置相机是否为单路录制

========================= TW02 =========================


========================= TW06 =========================

- getRecordConfigList(): 获取相机录制参数列表

- getCurRecordConfig(): 获取相机当前录制参数
- setCurRecordConfig(int index): 设置相机当前录制参数

- getForceCodec(): 获取相机是否强制H264编码
- setForceCodec(int force): 设置相机是否强制H264编码

- getVinMirrorList(): 获取相机镜头参数
- setVinMirrorList(List<String> lists): 设置相机镜头参数

========================= TW06 =========================

已完成

#### 数据接口

##### 连接机制，状态管理

已完成

##### 通信协议封装、解析

已完成

##### 数据接口封装

========================= TW02 =========================

- getClipPlaybackUrlWithStream(): 获取相机视频片段地址
- getClipDownloadInfo(): 获取相机视频片段下载信息

- deleteClip(): 删除相机视频片段
- addHighlightRx(): 标记相机视频片段

- getSingleClipRx(): 获取单个相机视频片段
- getRawDataBlockRx(): 获取一定范围内相机视频片段

- getSpaceInfo(): 获取相机存储信息

========================= TW02 =========================


========================= TW06 =========================

- getLiveDmsDataRx(boolean enable): 是否开启接收相机DMS数据

- doCalibrationRx(int x, int y, int z): 异步执行相机校验

========================= TW06 =========================

完成部分，尚未完成接口整合，预计还需一周

另外，如果有一些遗漏的点请补充。
