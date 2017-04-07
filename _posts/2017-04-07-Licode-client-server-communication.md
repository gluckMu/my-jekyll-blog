---
layout: post
title:  "Licode客户端服务器交互"
date:   2017-04-07
desc: "Licode客户端服务器交互"
keywords: "Licode"
categories: [WebRTC]
tags: [WebRTC]
---

# Licode客户端服务器交互

## WebRTC多人视频会议系统模型

基于WebRTC的多人视频会议系统主要有三种模型：Mesh，MCU和SFU。

### Mesh

在Mesh模型下，每一个参与者都与其他参与者单独建立一条连接，这就意味着n个用户的视频会议，每个参与者都有n-1条上行链路，以及n-1条下行链路。其优点是去中心化，服务器端压力小，因为其只需要处理Signal，后续的视频流都是直接走的P2P，同时由于每条视频流都是单独的，所以在客户端的处理更加自由，例如可以单独放大某个参与者的画面，但是缺点也异常明显，系统的可扩展性不高，并且对客户端的带宽要求非常高。这种模型更适合一对一的视频聊天。

<div align='center'>
<img src="{{site.baseurl}}{{ site.img_path }}/Licode/Mesh.png" alt="Mesh"/>
</div>

### MCU

MCU(Multipoint control unit)模型，要求每一个参与者都同一个中央控制节点去建立连接，由这个中央节点去负责将上传的多路视频流合并为一个，然后下发至每一个参与者，这样无论参与者有多少，每一个参与者的连接只有2条，一条上行，一条下行。其优点是系统的扩展性高，客户端的带宽压力小，缺点是中央节点的要求较高，需要将所有视频流合并，并且一旦有异常，就会导致整个系统不可用，另外，由于视频流在服务器端被合并，导致客户端无法对某一参与者的视频流进行处理。

<div align='center'>
<img src="{{site.baseurl}}{{ site.img_path }}/Licode/MCU.png" alt="MCU"/>
</div>

### SFU

SFU(Selective Forwarding Unit)模型，同MCU相同的是，都需要参与者同中央控制节点建立连接，但是与MCU不同的是，每一个参与者与中央节点的上行连接就1条，但是下行连接却是多条，每一条下行连接传输的是一个参与者的视频流，也就是说n个参与者的视频会议，每一个参与者都会有1条上行连接，n-1条下行连接。其优点是扩展性较好，客户端带宽压力较Mesh小，但是比MCU大，客户端可以处理单独的视频流，并且服务器端不再需要合并视频流，复杂性降低，缺点是服务器端的带宽压力大，相比于MCU，其下行链路增多。

<div align='center'>
<img src="{{site.baseurl}}{{ site.img_path }}/Licode/SFU.png" alt="SFU"/>
</div>

## Licode

### 概述

Licode是一个基于WebRTC的开源视频会议系统，采用的是SFU模型，Signal使用websokcet来实现。它包括了前端和后台两个部分，后台主要由C++开发的业务逻辑，使用Node.js封装出公共API供前端调用，前端只提供了Web版本，开发者可以根据个人需要，自行开发iOS或者Android等其他客户端版本，github上目前找到的只有iOS的客户端版本，但也有很多bug，需要自行解决，地址见[这里](https://github.com/zevarito/Licode-ErizoClientIOS)。

Licode官网已经介绍了其[架构](http://licode.readthedocs.io/en/master/)，包括Erizo、Erizo Room、Nuve API等，主要说明了后台的架构，鉴于我是一名前端工程师，关注的更多的是客户端与服务器的交互逻辑，因此后面主要介绍的是交互这块，其中大部分是WebRTC规范中提到的Signal机制，等到Singal结束后，剩下的视频流传输都是由WebRTC替我们实现了，还有一部分是Licode提供的room管理等功能。

### 客户端交互

#### 本地流上传

当本地需要加入房间或者创建房间时，首先需要一个房间名，这个房间名需要在第一步中使用。

1. 创建token，调用createToken接口，入参有role、room、username
2. 接口返回一串base64编码的json字符串，内容host——websocket连接地址，secure——是否使用https，tokenId——连接凭证，signature——签名。根据返回的地址，建立websocket连接（这一步实际上分为两个步骤，第一步是socket连接，然后返回一个类似于AjscRy-Q3Dx6N1l6l59R:60:60:websocket这样的返回，第一个值代表的是sid，websocket连接地址拼接上，第二个代表心跳时间，第三个代表连接超时时间，第四个代表传输协议，第二步根据传输协议再去升级websocket连接，但是这两步是由socketio直接完成的，所以上层并不知道）。
3. 等待websocket建立成功后，发送token消息，根据之前获得的token去获取roomId<br>
`{"name":"token","args":[{"signature":"N2E1NzRkMWQwZGRlOWM4MjNkZWU2MGJmMjFlNTdlMTBjM2YxMjE0Yw==","host":"45.77.0.189:8081","tokenId":"58e72d91c95d1d438a9f109e","secure":false}]}`

4. 接收token消息返回，第一个值代表是否成功，第二个值是个map，代表room信息，其中streams代表已发布的消息流，如果有的话就可以直接订阅，id代表房间号，如果是p2p模式，应该有个p2p的key，iceServers是webrtc可用的ice server列表，这个可以在后续建立RTCPeerConnection后设置。<br>
`["success",{"streams":[],"id":"58c7975b1a6acd1f81b9ed2e","defaultVideoBW":300,"maxVideoBW":300,"iceServers":[{"username":"tom","credential":"tom","url":"turn:104.198.87.161:8085"}]}]`

5. 根据上一步的返回，如果成功的话，发送发布消息，告诉服务器推送流信息，其中audio和video代表音视频，state有p2p和erizo两种，分别代表视频走p2p还是服务器，attributes是视频流相关信息，如果失败的话，结束流程。<br>
`{"name":"publish","args":[{"data":0,"audio":"true","state":"erizo","video":"true","attributes":{"name":"lili","actualName":"lili","type":"public"}},"null"]}`

6. 等待发布消息返回，返回的值就是本地流的streamId。返回成功之后，就可以创建RTCPeerConnection了，之前收到的iceServers，这时候就可以设置进去了，并将本地视频流添加到peerConnection，创建sdp消息，从这开始，就是WebRTC的Signal流程。可能之后还会收到个start消息，代表流开始。<br>
`[219328436604859780]`
`
{"name":"signaling_message_erizo","args":[{"mess":{"type":"started"},"streamId":219328436604859780}]}
`

7. sdp创建结束后，如果失败，结束流程，如果成功，然后就调用peerConnection的setLocalDescription，设置本地sdp。

8. 设置成功后，发送sdp消息。<br>
```
{"name":"signaling_message","args":[{"msg":{"type":"offer","sdp":"v=0\r\no=- 8850032263846546616 2 IN IP4 127.0.0.1\r\ns=-\r\nt=0 0\r\na=group:BUNDLE audio video\r\na=msid-semantic: WMS LCMSv0\r\nm=audio 9 UDP\/TLS\/RTP\/SAVPF 111 103 104 9 102 0 8 106 105 13 126\r\nc=IN IP4 0.0.0.0\r\na=rtcp:9 IN IP4 0.0.0.0\r\na=ice-ufrag:PfiA\r\na=ice-pwd:WVTeyTehg9yIdBNyuCyrXzSB\r\na=ice-options:renomination\r\na=fingerprint:sha-256 9A:6E:BE:07:68:C1:44:A2:54:FD:48:C9:6C:51:8B:F0:1D:7B:EF:51:1C:23:03:40:06:7F:DC:5F:2B:27:FC:2E\r\na=setup:actpass\r\na=mid:audio\r\na=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level\r\na=sendrecv\r\na=rtcp-mux\r\na=rtpmap:111 opus\/48000\/2\r\na=rtcp-fb:111 transport-cc\r\na=fmtp:111 minptime=10;useinbandfec=1\r\na=rtpmap:103 ISAC\/16000\r\na=rtpmap:104 ISAC\/32000\r\na=rtpmap:9 G722\/8000\r\na=rtpmap:102 ILBC\/8000\r\na=rtpmap:0 PCMU\/8000\r\na=rtpmap:8 PCMA\/8000\r\na=rtpmap:106 CN\/32000\r\na=rtpmap:105 CN\/16000\r\na=rtpmap:13 CN\/8000\r\na=rtpmap:126 telephone-event\/8000\r\na=ssrc:1759223096 cname:B+owTYyjoYX\/L0P1\r\na=ssrc:1759223096 msid:LCMSv0 LCMSa0\r\na=ssrc:1759223096 mslabel:LCMSv0\r\na=ssrc:1759223096 label:LCMSa0\r\nm=video 9 UDP\/TLS\/RTP\/SAVPF 120 100 101 116 117 96 97 98 99\r\nc=IN IP4 0.0.0.0\r\na=rtcp:9 IN IP4 0.0.0.0\r\na=ice-ufrag:PfiA\r\na=ice-pwd:WVTeyTehg9yIdBNyuCyrXzSB\r\na=ice-options:renomination\r\na=fingerprint:sha-256 9A:6E:BE:07:68:C1:44:A2:54:FD:48:C9:6C:51:8B:F0:1D:7B:EF:51:1C:23:03:40:06:7F:DC:5F:2B:27:FC:2E\r\na=setup:actpass\r\na=mid:video\r\na=extmap:2 urn:ietf:params:rtp-hdrext:toffset\r\na=extmap:3 http:\/\/www.webrtc.org\/experiments\/rtp-hdrext\/abs-send-time\r\na=extmap:4 urn:3gpp:video-orientation\r\na=extmap:5 http:\/\/www.ietf.org\/id\/draft-holmer-rmcat-transport-wide-cc-extensions-01\r\na=extmap:6 http:\/\/www.webrtc.org\/experiments\/rtp-hdrext\/playout-delay\r\na=sendrecv\r\na=rtcp-mux\r\na=rtcp-rsize\r\na=rtpmap:100 VP8\/90000\r\na=rtcp-fb:100 ccm fir\r\na=rtcp-fb:100 nack\r\na=rtcp-fb:100 nack pli\r\na=rtcp-fb:100 goog-remb\r\na=rtcp-fb:100 transport-cc\r\na=rtpmap:101 VP9\/90000\r\na=rtcp-fb:101 ccm fir\r\na=rtcp-fb:101 nack\r\na=rtcp-fb:101 nack pli\r\na=rtcp-fb:101 goog-remb\r\na=rtcp-fb:101 transport-cc\r\na=rtpmap:116 red\/90000\r\na=rtpmap:117 ulpfec\/90000\r\na=rtpmap:120 H264\/90000\r\na=rtcp-fb:120 ccm fir\r\na=rtcp-fb:120 nack\r\na=rtcp-fb:120 nack pli\r\na=rtcp-fb:120 goog-remb\r\na=rtcp-fb:120 transport-cc\r\na=fmtp:120 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42e01f\r\na=rtpmap:96 rtx\/90000\r\na=fmtp:96 apt=100\r\na=rtpmap:97 rtx\/90000\r\na=fmtp:97 apt=101\r\na=rtpmap:98 rtx\/90000\r\na=fmtp:98 apt=116\r\na=rtpmap:99 rtx\/90000\r\na=fmtp:99 apt=120\r\na=ssrc-group:FID 1875410319 2262661321\r\na=ssrc:1875410319 cname:B+owTYyjoYX\/L0P1\r\na=ssrc:1875410319 msid:LCMSv0 LCMSv0\r\na=ssrc:1875410319 mslabel:LCMSv0\r\na=ssrc:1875410319 label:LCMSv0\r\na=ssrc:2262661321 cname:B+owTYyjoYX\/L0P1\r\na=ssrc:2262661321 msid:LCMSv0 LCMSv0\r\na=ssrc:2262661321 mslabel:LCMSv0\r\na=ssrc:2262661321 label:LCMSv0\r\n"},"streamId":"219328436604859780"},"null"]}
```

9. 等待接收answer消息，接收到answer消息，根据该消息生成远端的sdp，然后调用RTCPeerConnection的setRemoteDescription设置远端的sdp。<br>
`{"name":"signaling_message_erizo","args":[{"mess":{"type":"answer","sdp":"v=0\no=- 0 0 IN IP4 127.0.0.1\ns=LicodeMCU\nt=0 0\na=group:BUNDLE audio video\na=msid-semantic: WMS fa37JncCHr\nm=audio 1 UDP/TLS/RTP/SAVPF 0 126\nc=IN IP4 0.0.0.0\na=rtcp:1 IN IP4 0.0.0.0\na=candidate:2 1 udp 2013266431 45.77.0.189 41526 typ host generation 0\na=ice-ufrag:+Fki\na=ice-pwd:7sH3MsRhJiTUp68pl468fM\na=fingerprint:sha-256 5C:6A:D9:39:36:45:8E:51:F7:81:9D:39:59:F0:DE:73:AF:DC:17:DE:D3:3B:9F:3F:61:5C:8D:D6:4A:17:B3:27\na=setup:active\na=sendrecv\na=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level\na=mid:audio\na=rtcp-mux\na=rtpmap:0 PCMU/8000\na=rtpmap:126 telephone-event/8000\na=ssrc:44444 cname:o/i14u9pJrxRKAsu\na=ssrc:44444 msid:fa37JncCHr a0\na=ssrc:44444 mslabel:fa37JncCHr\na=ssrc:44444 label:fa37JncCHra0\nm=video 1 UDP/TLS/RTP/SAVPF 100\nc=IN IP4 0.0.0.0\na=rtcp:1 IN IP4 0.0.0.0\na=candidate:2 1 udp 2013266431 45.77.0.189 41526 typ host generation 0\na=ice-ufrag:+Fki\na=ice-pwd:7sH3MsRhJiTUp68pl468fM\na=extmap:2 urn:ietf:params:rtp-hdrext:toffset\na=extmap:3 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time\na=extmap:4 urn:3gpp:video-orientation\na=extmap:6 http://www.webrtc.org/experiments/rtp-hdrext/playout-delay\na=fingerprint:sha-256 5C:6A:D9:39:36:45:8E:51:F7:81:9D:39:59:F0:DE:73:AF:DC:17:DE:D3:3B:9F:3F:61:5C:8D:D6:4A:17:B3:27\na=setup:active\na=sendrecv\na=mid:video\na=rtcp-mux\na=rtpmap:100 VP8/90000\na=rtcp-fb:100 ccm fir\na=rtcp-fb:100 nack\na=rtcp-fb:100 nack pli\na=rtcp-fb:100 goog-remb\na=ssrc:55543 cname:o/i14u9pJrxRKAsu\na=ssrc:55543 msid:fa37JncCHr v0\na=ssrc:55543 mslabel:fa37JncCHr\na=ssrc:55543 label:fa37JncCHrv0\n"},"streamId":219328436604859780}]}`

10. 当远端sdp设置成功后，WebRTC的Signal就结束了，剩下的WebRTC就会自己去传输视频流等。

11. 接收到onAddStream消息，streamId就是本地流的streamId，代表本地流发布成功。<br>
`{"name":"onAddStream","args":[{"id":219328436604859780,"audio":"true","video":"true","data":0,"attributes":{"name":"lili","actualName":"lili","type":"public"}}]}`

12. 接收到ready消息，代表一切就绪。<br>
`{"name":"signaling_message_erizo","args":[{"mess":{"type":"ready"},"streamId":219328436604859780}]}`

#### Candidate

当设置完本地sdp后，WebRTC就会开始产生IceCandidate，并调用回调，这时候我们将生成的IceCandidate保存起来，等到RTCIceGatheringStateComplete通知后再逐一发送，也可以生成一个就发送一个，但是需要注意的是，发送IceCandidate最好要保证远端的RTCPeerConnection创建成功，否则对方就无法设置了。<br>
`{"name":"signaling_message","args":[{"msg":{"type":"candidate","candidate":{"sdpMLineIndex":0,"sdpMid":"audio","candidate":"a=candidate:2400357177 1 udp 2122262783 2409:8800:9210:204a:8f1:576a:f5a1:b350 65079 typ host generation 0 ufrag PfiA network-id 4 network-cost 50"}},"streamId":"219328436604859780"},"null"]}`

#### 订阅远端流

当远端一个新用户加入房间时，已有的用户就需要开始订阅流程。

1. 接收到onAddStream消息，告知有新用户加入。<br>
`{"name":"onAddStream","args":[{"id":275079030538886180,"audio":true,"video":true,"data":true,"screen":""}]}`

2. 根据接收的streamId，发送订阅消息。<br>
`{"name":"subscribe","args":[{"streamId":"275079030538886180"},"null"]}`

3. 等待订阅消息返回，如果返回结果为true代表可以订阅，否则不能订阅。如果能够订阅的话，就同发布消息类似，创建RTCPeerConnection，设置IceServers，但是不需要添加本地流，创建sdp消息。这一步，和发布消息一样，还会收到start消息。<br>
`[true]`
`{"name":"signaling_message_erizo","args":[{"mess":{"type":"started"},"peerId":"275079030538886180"}]}`

4. 创建sdp成功后，设置本地sdp，然后发送sdp消息。

5. 接收answer消息，设置远端sdp，接收ready消息，至此同发布的后续流程都一致。

6. 在设置完远端sdp后，WebRTC的Signal就结束了，后续就会在RTCPeerConnection的回调中，调用didAddStream方法，意味着远端流的到来，如果需要将远端展示出来，可以在这个回调中进行处理。

#### 移除远端流

当远端用户退出房间时，已有用户需要移除掉该远端流。

1. 接收到onRemoveStream消息，告知有用户退出。<br>
`{"name":"onRemoveStream","args":[{"id":275079030538886180}]}`

2. 移除相关展示，关闭RTCPeerConnection。


#### 其他

Websocket接收的消息还有offer，candidate，以及bye消息，Licode本身支持P2P模式，但是在实际中还是得经过服务器去实现，因为需要视频存档之类的，如果只是纯粹的走SFU这种模式的话，每次都是由客户端去创建RTCPeerConnection，并且发送offer给服务器，服务器返回answer，客户顿理论上是不会接收到offer消息的。Candidate消息，理论上可以接收到，但是在实际测试中，并没有发现Licode发送过candidate消息，只有在answer中包含了candidate内容，应该是Licode处理过。Bye消息就是直接断掉连接就好。


图片源自http://testrtc.com/different-multiparty-video-conferencing/