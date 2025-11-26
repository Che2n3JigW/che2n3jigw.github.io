---
categories: Android
title: WebRTC 接收端连接流程
toc: true
toc_sticky: true
---

{% 
include figure popup=true image_path="/assets/images/android/webrtc/connect.jpg" 
%}

在当前的流程图中，**发送端的流程被弱化处理，主要聚焦于接收端的流程。**

关于类设计方面，目前有两个主要类：

SignalingHandler：负责信令相关功能，例如与信令服务器交互、处理远端 SDP 和 ICE Candidate。专注于 信令层（消息传递、事件通知）。

PeerConnectionHandler：负责 WebRTC 连接管理，例如创建 PeerConnection、添加轨道、监听远端媒体流。专注于 WebRTC 层（连接管理、媒体处理）

## 初始化阶段

1. **连接信令服务器**：建立与信令服务器的通信通道，用于交换 SDP 与 ICE Candidate。
2. **创建 `PeerConnection` 实例**：初始化 WebRTC 核心连接对象，配置 ICE 服务器等参数。

## 接收远端 Offer

1. **等待远端 Offer SDP**（信令服务器通知）：接收房间中已有用户发来的 Offer。
2. **设置远端描述**：调用 `peerConnection.setRemoteDescription(offer)` 将远端 SDP 应用到本地。

## 创建本地 Answer

1. **生成 Answer SDP**：调用 `peerConnection.createAnswer()` 生成本地 Answer。
2. **设置本地描述**：调用 `peerConnection.setLocalDescription(answer)`。
3. **通过信令服务器发送 Answer SDP 到远端**。

## ICE Candidate 处理

1. **等待远端 ICE Candidate**（信令服务器通知）。
2. **添加远端 ICE Candidate**：调用 `peerConnection.addIceCandidate(candidate)`。
3. **监听本地 ICE Candidate 生成事件**：通过 `PeerConnection.Observer.onIceCandidate` 回调捕获本地候选。
4. **通过信令服务器发送本地 ICE Candidate 到远端**。

## 远端媒体轨道处理

1. **监听远端轨道**：通过 `PeerConnection.Observer.onTrack` 回调获取远端 MediaStream。
2. **绑定播放控件**：将远端轨道添加到 `SurfaceViewRenderer` 播放。