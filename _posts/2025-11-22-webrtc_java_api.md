---
categories: Android
title: WebRTC Java API 讲解
toc: true
toc_sticky: true
---





# PeerConnectionFactory

它的作用是为整个 WebRTC 系统创建和管理底层资源，包括 **PeerConnection、音视频轨道、数据通道、音视频源** 等。

| 方法                                               | 作用               | 备注                 |
| -------------------------------------------------- | ------------------ | -------------------- |
| `initialize(InitializationOptions)`                | 初始化 WebRTC 环境 | 必须先调用           |
| `createPeerConnection(RTCConfiguration, Observer)` | 创建 P2P 连接      | 最常用，支持多种重载 |
| `createLocalMediaStream(String)`                   | 创建本地流         | 用于音视频轨道绑定   |
| `createAudioSource(MediaConstraints)`              | 创建音频源         | 可设置约束           |
| `createAudioTrack(String, AudioSource)`            | 创建音频轨道       | 绑定源生成轨道       |
| `createVideoSource(boolean)`                       | 创建视频源         | 可设置是否屏幕共享   |
| `createVideoTrack(String, VideoSource)`            | 创建视频轨道       | 绑定源生成轨道       |
| `getRtpSenderCapabilities(MediaType)`              | 查询发送端能力     | 用于编码/解码选择    |
| `getRtpReceiverCapabilities(MediaType)`            | 查询接收端能力     | 同上                 |
| `startInternalTracingCapture(String)`              | 开启调试追踪       | 可选                 |
| `stopInternalTracingCapture()`                     | 关闭调试追踪       | 可选                 |
| `dispose()`                                        | 释放资源           | 用完必须调用         |

## InitializationOptions

`PeerConnectionFactory.InitializationOptions` 类及其内部的 `Builder` 类，它主要是 **用来配置和初始化 WebRTC Android SDK 的全局选项**。这是初始化 `PeerConnectionFactory` 的必要步骤之一。

| 字段                   | 类型                       | 作用                                                         |
| ---------------------- | -------------------------- | ------------------------------------------------------------ |
| `applicationContext`   | `Context`                  | Android 应用上下文，必须提供，WebRTC 内部需要访问系统服务、加载库等 |
| `fieldTrials`          | `String`                   | 实验性特性开关（field trials），可以控制 WebRTC 内部功能启用或禁用 |
| `enableInternalTracer` | `boolean`                  | 是否启用内部追踪器，用于性能分析和调试                       |
| `nativeLibraryLoader`  | `NativeLibraryLoader`      | 自定义原生库加载器，如果你有特殊路径或加载方式可以指定       |
| `nativeLibraryName`    | `String`                   | 原生库名称，默认 `"jingle_peerconnection_so"`                |
| `loggable`             | `Loggable`（可空）         | 自定义日志接收器，可以把 WebRTC 日志输出到自己的系统         |
| `loggableSeverity`     | `Logging.Severity`（可空） | 日志级别，决定输出哪些日志                                   |

## Builder

`PeerConnectionFactory.Builder` 类是 **WebRTC Android SDK 中构建 `PeerConnectionFactory` 的核心构造器**，它负责将各种可选配置（音视频编解码、音频设备模块、网络控制器等）组装成一个完整的工厂对象。

| 字段                                  | 类型                                   | 作用                                                         |
| ------------------------------------- | -------------------------------------- | ------------------------------------------------------------ |
| `options`                             | `Options`                              | PeerConnectionFactory 的全局选项，如禁用加密、禁用网络监控等 |
| `audioDeviceModule`                   | `AudioDeviceModule`                    | 音频输入/输出模块，如果为空，会使用默认 `JavaAudioDeviceModule` |
| `audioEncoderFactoryFactory`          | `AudioEncoderFactoryFactory`           | 音频编码器工厂                                               |
| `audioDecoderFactoryFactory`          | `AudioDecoderFactoryFactory`           | 音频解码器工厂                                               |
| `videoEncoderFactory`                 | `VideoEncoderFactory`                  | 视频编码器                                                   |
| `videoDecoderFactory`                 | `VideoDecoderFactory`                  | 视频解码器                                                   |
| `audioProcessingFactory`              | `AudioProcessingFactory`               | 音频处理（AEC、AGC、NS）                                     |
| `fecControllerFactoryFactory`         | `FecControllerFactoryFactoryInterface` | FEC 前向纠错控制器                                           |
| `networkControllerFactoryFactory`     | `NetworkControllerFactoryFactory`      | 网络控制器工厂                                               |
| `networkStatePredictorFactoryFactory` | `NetworkStatePredictorFactoryFactory`  | 网络状态预测器工厂                                           |
| `neteqFactoryFactory`                 | `NetEqFactoryFactory`                  | 音频解码和同步工厂                                           |

## Options

`PeerConnectionFactory.Options` 类是 **WebRTC Android SDK 中配置 PeerConnectionFactory 行为的辅助类**，它主要用于在 **创建 PeerConnectionFactory 时传入的一些全局可选配置**。相比 `Builder` 那么复杂，它的作用比较单一，主要是对网络和加密的一些开关控制。

| 字段                    | 类型      | 默认值 | 作用                                                         |
| ----------------------- | --------- | ------ | ------------------------------------------------------------ |
| `networkIgnoreMask`     | `int`     | 0      | 用位掩码控制忽略的网络类型，例如：`ADAPTER_TYPE_WIFI` 表示忽略 Wi-Fi 网络 |
| `disableEncryption`     | `boolean` | false  | 是否禁用 SRTP 加密，通常不建议关闭                           |
| `disableNetworkMonitor` | `boolean` | false  | 是否禁用 WebRTC 内部网络状态监控（网络变化监听）             |



# PeerConnection

## RTCConfiguration

`PeerConnection.RTCConfiguration` 类是 **WebRTC Android SDK 中配置 PeerConnection 的核心类**，它定义了 **ICE、网络策略、SDP、端口和加密等各种连接参数**。简单说，它决定了 PeerConnection 如何建立、管理和优化 P2P 连接。

| 字段                          | 类型                     | 作用                                               |
| ----------------------------- | ------------------------ | -------------------------------------------------- |
| `iceServers`                  | `List<IceServer>`        | ICE Server 列表（STUN/TURN），必填                 |
| `iceTransportsType`           | `IceTransportsType`      | 控制使用的候选类型（ALL / RELAY / NOHOST 等）      |
| `bundlePolicy`                | `BundlePolicy`           | 是否将音视频轨道捆绑在同一个传输通道，减少端口占用 |
| `rtcpMuxPolicy`               | `RtcpMuxPolicy`          | 是否捆绑 RTP/RTCP                                  |
| `tcpCandidatePolicy`          | `TcpCandidatePolicy`     | 是否允许 TCP 作为候选传输                          |
| `candidateNetworkPolicy`      | `CandidateNetworkPolicy` | 控制允许使用的网络类型（ALL / LOW_COST 等）        |
| `audioJitterBufferMaxPackets` | `int`                    | 音频抖动缓冲区大小                                 |
| `enableCpuOveruseDetection`   | `boolean`                | 是否启用 CPU 过载检测，动态调节视频码率            |
| `sdpSemantics`                | `SdpSemantics`           | SDP 格式（Plan B / Unified Plan）                  |
| `iceCandidatePoolSize`        | `int`                    | 预生成 ICE 候选数量，提高连接速度                  |
| `turnPortPrunePolicy`         | `PortPrunePolicy`        | TURN 端口修剪策略                                  |
| `networkPreference`           | `AdapterType`            | 网络首选类型（Wi-Fi / Cellular / Any）             |
| `disableIPv6OnWifi`           | `boolean`                | Wi-Fi 禁用 IPv6                                    |
| `cryptoOptions`               | `CryptoOptions`          | SRTP 高级选项                                      |

## 
