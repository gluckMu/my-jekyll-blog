---
layout: post
title:  "直播调研"
date:   2017-08-11
desc: "直播调研"
keywords: "直播调研"
categories: [直播]
tags: [直播]
---

## 分类

* UGC，User-generated Content也称为UCC，User-created Content
* PGC，Professionally-generated Content也称为PPC，Professionally-produced Content
* OGC，Occupationally-generated Content

<div align='center'>
<img src="{{site.baseurl}}{{ site.img_path }}/Live_Introduction/category.png" alt="直播分类"/>
</div>

不同类型的直播导致需求的不同，例如连麦，这个会影响技术的实现。

## 功能

* 媒体模块：直播、点播，包括推流端（采集、前处理、编码、推流），服务端处理（转码、录制、截图、鉴黄），播放器（拉流、解码、渲染）
* 服务模块：礼物、支付、运营、任务、IM（弹幕）、安全、统计
* 其他

具体的各模块介绍，可以参考[七牛云视频直播技术详解系列](https://www.zhihu.com/question/42162310)

## 协议

### RTP

Real-time Transport Protocol，用于Internet上针对多媒体数据流的一种传输层协议。实际应用场景下经常需要RTCP（RTP Control Protocol）配合来使用，可以简单理解为RTCP传输交互控制的信令，RTP传输实际的媒体数据。

RTP使用UDP协议来传输数据，导致其实时性较强，但是可靠性就得不到保障，因此其在视频监控、视频会议、IP电话上有广泛的应用。

### RTMP

Real Time Messaging Protocol，实时消息传送协议。RTMP是 Adobe Systems 公司为 Flash 播放器和服务器之间音频、视频和数据传输开发的开放协议。协议基于 TCP，是一个协议族，包括 RTMP 基本协议及 RTMPT/RTMPS/RTMPE 等多种变种。

RTMP 是一种设计用来进行实时数据通信的网络协议，主要用来在 Flash/AIR 平台和支持RTMP协议的流媒体/交互服务器之间进行音视频和数据通信。RTMP直播延迟在1-3秒，CDN支持较好。

### HLS

HTTP Live Streaming，是苹果公司实现的基于 HTTP 的流媒体传输协议，可支持流媒体的直播和点播，主要应用在 iOS 系统，为 iOS 设备（如 iPhone、iPad）提供音视频直播和点播方案。HLS直播延迟在10s，适用于点播系统。

### HTTP-FLV

HTTP-FLV，是指将音视频数据封装成flv格式，然后通过HTTP协议传输到客户端，由于FLV 协议中, 每一个音视频数据都被封装成了包含时间戳信息头的数据包. 当播放器拿到这些数据包解包的时候能够根据时间戳信息把这些音视频数据和之前到达的音视频数据连续起来播放。

HTTP-FLV采用HTTP协议，这就导致它相比于RTMP，有以下优点：

* 可以在一定程度上避免防火墙的干扰 (例如, 有的机房只允许 80 端口通过).
* 可以很好的兼容 HTTP 302 跳转, 做到灵活调度.
* 可以使用 HTTPS 做加密通道 

同时，HTTP-FLV在延迟上也基本与RTMP相同，所以在直播的下行方面，可以采用此协议。

### WebRTC

Web Real-Time Communication，网页即时通信，它由一组标准、协议和 JavaScript API 组成，用于实现端到端的音视频及数据共享，他并不是协议，而是一套解决方案。与其他浏览器通信机制不同，WebRTC 通过 RTP 传输数据，延迟在毫秒级，适用于实时的音视频交互场景。

WebRTC很适合用在网络电话这种需要双向视频通话的场景上，也支持直播这种一对多的场景，但是比较适合小范围的，连麦时就可以使用。

## 开源项目

### 服务器端

* [SRS](https://github.com/ossrs/srs)：国产独立的RTMP服务
* [NGINX-RTMP](https://github.com/arut/nginx-rtmp-module)：基于NGINX的rtmp服务模块，需要结合nginx来进行编译
* 各家厂商提供的直播云
* 其他开源（EasyDarwin等)

### iOS

#### 推流端

* [LFLiveKit](https://github.com/LaiFengiOS/LFLiveKit)：优酷旗下来疯直播开源
* [LiveVideoCoreSDK](https://github.com/runner365/LiveVideoCoreSDK)：基于开源videocore进行了改进
* [AnyRTC](https://github.com/AnyRTC/anyRTC-RTMP-OpenSource)：实现推流端以及播放端，支持连麦，跨平台（Win,IOS,Android）
* 各种厂商提供的sdk，例如[七牛](https://github.com/pili-engineering/PLMediaStreamingKit)

#### 播放端

* [ijkplayer](https://github.com/Bilibili/ijkplayer/)：Bilibili开源的多媒体播放器

### Android

#### 推流端

* [yasea](https://github.com/begeekmyfriend/yasea)

#### 播放端

* [ijkplayer](https://github.com/Bilibili/ijkplayer/)：Bilibili开源的多媒体播放器

### Windows

#### 推流端

* [OBS](https://github.com/jp9000/OBS)
* [biliobs](https://github.com/Bilibili/biliobs)：Bilibili开源，基于OBS二次开发

#### 播放端

直接浏览器即可

## 参考

* [知乎](https://www.zhihu.com/question/42162310)


