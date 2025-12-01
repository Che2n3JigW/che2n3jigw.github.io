---
categories: Android
tags: webrtc
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

`PeerConnection` 是 WebRTC 的核心类，代表一个点对点连接，负责：

- ICE 连接建立
- SDP 协商
- 媒体流的发送与接收
- 数据通道
- RTP（Sender、Receiver、Transceiver）
- 状态管理
- 日志和事件记录
- 资源释放



1. 成员变量

   | 字段                   | 作用                                                 |
   | ---------------------- | ---------------------------------------------------- |
   | `localStreams`         | 保存 addStream() 添加的本地 MediaStream（仅 Plan B） |
   | `nativePeerConnection` | C++ 层的 PeerConnection 指针（long）                 |
   | `senders`              | 当前所有 RtpSender （发送音视频轨道）                |
   | `receivers`            | 当前所有 RtpReceiver（接收音视频轨道）               |
   | `transceivers`         | RtpTransceiver 列表（Unified Plan）                  |

2. 构造函数

   | 构造方式                                    | 用途                               |
   | ------------------------------------------- | ---------------------------------- |
   | 通过 `factory.createNativePeerConnection()` | 正常 Java 创建                     |
   | 直接传入 long 指针                          | JNI 手动创建 PeerConnection 时使用 |

3. SDP & 描述管理（JSEP）

   | 方法                     | 作用            |
   | ------------------------ | --------------- |
   | `getLocalDescription()`  | 获取本地 SDP    |
   | `getRemoteDescription()` | 获取远端 SDP    |
   | `createOffer()`          | 创建 SDP offer  |
   | `createAnswer()`         | 创建 SDP answer |
   | `setLocalDescription()`  | 设置本地 SDP    |
   | `setRemoteDescription()` | 设置远端 SDP    |

4. ICE 操作

   | 方法                    | 功能               |
   | ----------------------- | ------------------ |
   | `restartIce()`          | 重启 ICE           |
   | `addIceCandidate()`     | 添加 ICE candidate |
   | `removeIceCandidates()` | 移除多个 candidate |

5. 音频控制

   | 方法                         | 作用              |
   | ---------------------------- | ----------------- |
   | `setAudioPlayout(boolean)`   | 开启/关闭播放音频 |
   | `setAudioRecording(boolean)` | 开启/关闭采集音频 |

6. 配置 RTCConfiguration

   **setConfiguration**

   动态修改 ICE server、bundlePolicy 等。

7. MediaStream（Plan B 专用）

   | 方法                        | 说明                     |
   | --------------------------- | ------------------------ |
   | `addStream(MediaStream)`    | 添加整个媒体流（Plan B） |
   | `removeStream(MediaStream)` | 移除媒体流               |

   Unified Plan 已弃用 addStream/removeStream。

8. 发送轨道相关（RtpSender）

   **createSender**

   Plan B 的方法，用于预先创建一个 sender（不带 track）。

   **getSenders**

   获取当前 senders,每次调用会 dispose 老的 sender，再重新从 native 获取。

9. 接收轨道（RtpReceiver）

   同 getSenders 一样，有 `getReceivers()`。

10. Transceivers（Unified Plan）

    | 方法                            | 说明         |
    | ------------------------------- | ------------ |
    | addTransceiver(track)           | 基于 track   |
    | addTransceiver(track, init)     | track + 配置 |
    | addTransceiver(mediaType)       | AUDIO/VIDEO  |
    | addTransceiver(mediaType, init) | 类型 + 配置  |

11. 添加/移除 Track

    **addTrack**

    **removeTrack**

12. Stats 统计

    ```java
    getStats(RTCStatsCollectorCallback)
    getStats(RtpSender, RTCStatsCollectorCallback)
    getStats(RtpReceiver, RTCStatsCollectorCallback)
    ```

13. 设置 RTP 带宽

    ```java
    setBitrate(min, current, max)
    ```

14. RTC EventLog

    | 方法                       | 说明     |
    | -------------------------- | -------- |
    | startRtcEventLog(fd, size) | 开始记录 |
    | stopRtcEventLog()          | 停止记录 |

15. 状态查询

    | 方法                   | 对应状态                         |
    | ---------------------- | -------------------------------- |
    | `signalingState()`     | stable / have-local-offer 等     |
    | `iceConnectionState()` | checking / connected / failed 等 |
    | `connectionState()`    | new → connecting → connected     |
    | `iceGatheringState()`  | new → gathering → complete       |

16. close() & dispose()

    close()

    关闭连接，但不释放 Java/C++ 对象。

    dispose()

    完全释放所有资源：

    - close()
    - dispose 所有 stream/sender/receiver/transceiver
    - 清空列表
    - 调用 `nativeFreeOwnedPeerConnection`

    必须在所有操作完成后调用。



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

## Observer 

用于监听 WebRTC PeerConnection 的所有状态变化，包括信令、ICE、媒体轨道、数据通道、连接状态等。

| 方法                                                         | 作用                                  | 备注                                   |
| ------------------------------------------------------------ | ------------------------------------- | -------------------------------------- |
| **onSignalingChange(SignalingState)**                        | 信令状态变化（如 stable、have-offer） | 由 SDP 状态机驱动                      |
| **onIceConnectionChange(IceConnectionState)**                | ICE 连接状态变化，如 CONNECTED/FAILED | 网络变化时常触发                       |
| **onStandardizedIceConnectionChange(IceConnectionState)**    | WebRTC 新标准下的 ICE 状态变化        | 默认空实现                             |
| **onConnectionChange(PeerConnectionState)**                  | PeerConnection 整体连接状态变化       | 高层级连接状态（NEW/CONNECTED/FAILED） |
| **onIceConnectionReceivingChange(boolean)**                  | 指示 ICE 是否正在接收数据             | 可用于判断卡顿或网络断流               |
| **onIceGatheringChange(IceGatheringState)**                  | ICE candidate 收集状态变化            | NEW/GATHERING/COMPLETE                 |
| **onIceCandidate(IceCandidate)**                             | 本地发现新 Candidate                  | 需通过信令发给对端                     |
| **onIceCandidateError(IceCandidateErrorEvent)**              | ICE candidate 收集失败                | 默认空实现，开发者可监听错误           |
| **onIceCandidatesRemoved(IceCandidate[])**                   | 某些 Candidate 被废弃/删除            | 与对端同步 Candidate 状态              |
| **onSelectedCandidatePairChanged(CandidatePairChangeEvent)** | 当前使用的 Candidate pair 修改        | 网络切换时触发（如 WiFi→4G）           |
| **onAddStream(MediaStream)**                                 | 远端增加一个 MediaStream（Plan-B）    | 旧版模式使用                           |
| **onRemoveStream(MediaStream)**                              | 远端移除一个 MediaStream（Plan-B）    | 旧版模式使用                           |
| **onDataChannel(DataChannel)**                               | 收到远端创建的 DataChannel            | 只有对端 create 时触发                 |
| **onRenegotiationNeeded()**                                  | 需要重新协商（offer/answer）          | 添加 track/transceiver 时触发          |
| **onAddTrack(RtpReceiver, MediaStream[])**                   | 远端添加 Track（Unified Plan）        | 新版 WebRTC 推荐                       |
| **onRemoveTrack(RtpReceiver)**                               | 远端移除 Track                        | 对应 setRemoteDescription              |
| **onTrack(RtpTransceiver)**                                  | 收到远端发送的 Transceiver            | Unified Plan 本质回调                  |

## **IceServer**

`PeerConnection.IceServer` 表示一个 STUN / TURN 服务器的配置，包含其地址、认证方式以及 TLS 安全相关设置。
 WebRTC 会根据 IceServer 信息去收集 ICE 候选者，实现 NAT 穿透。

| 字段                | 类型               | 作用                                             |
| ------------------- | ------------------ | ------------------------------------------------ |
| `uri`               | `String`（已弃用） | 旧版单一 URL，已弃用，使用 `urls` 代替           |
| `urls`              | `List<String>`     | 支持多个 STUN/TURN URL（RFC7064/7065 格式）      |
| `username`          | `String`           | TURN 授权用户名                                  |
| `password`          | `String`           | TURN 授权密码                                    |
| `tlsCertPolicy`     | `TlsCertPolicy`    | TLS 证书校验策略                                 |
| `hostname`          | `String`           | 主机名（用于 TLS SNI，当 URL 使用 IP 时可指定）  |
| `tlsAlpnProtocols`  | `List<String>`     | TLS ALPN 协议列表，可为空                        |
| `tlsEllipticCurves` | `List<String>`     | TLS 握手椭圆曲线列表（如 `"P-256"`, `"X25519"`） |

| 方法                                                         | 作用                        | 备注             |
| ------------------------------------------------------------ | --------------------------- | ---------------- |
| `IceServer(String uri)`                                      | 创建只包含一个 URL 的服务器 | **已弃用**       |
| `IceServer(String uri, String username, String password)`    | 旧版服务器构造，带账号密码  | **已弃用**       |
| `IceServer(String uri, String username, String password, TlsCertPolicy tlsCertPolicy)` | 旧版构造，加入证书策略      | **已弃用**       |
| `IceServer(String uri, String username, String password, TlsCertPolicy tlsCertPolicy, String hostname)` | 加入 hostname（SNI）        | **已弃用**       |
| `private IceServer(...)`                                     | 全字段构造（内部用）        | Builder 最终调用 |

### Builder

| 方法                     | 作用                      | 备注          |
| ------------------------ | ------------------------- | ------------- |
| `builder(String uri)`    | 用单一 URL 创建 Builder   | 最终用 `urls` |
| `builder(List<String>)`  | 用多个 URL 创建 Builder   | 推荐方式      |
| `setUsername(String)`    | 设置 TURN 用户名          | 可为空        |
| `setPassword(String)`    | 设置 TURN 密码            | 可为空        |
| `setTlsCertPolicy()`     | 设置 TLS 证书策略         | 默认安全      |
| `setHostname(String)`    | 设置 SNI 主机名           | 用于 TLS      |
| `setTlsAlpnProtocols()`  | 设置 ALPN 协议            | 可选          |
| `setTlsEllipticCurves()` | 设置椭圆曲线              | 可选          |
| `createIceServer()`      | 创建最终的 IceServer 对象 | Builder 结束  |
