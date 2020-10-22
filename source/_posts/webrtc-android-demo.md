---
title: WebRTC实现Android传屏demo
date: 2020-10-19 21:00:23
categories: WebRTC
tags: [Android, WebRTC, Screen Share]
---

## WebRTC简介

WebRTC (Web Real-Time Communications) 是一项实时通讯技术，它允许网络应用或者站点，在不借助中间媒介的情况下，建立浏览器之间点对点（Peer-to-Peer）的连接，实现视频流和音频流或其他任意数据的传输。

目前，WebRTC的应用已经不局限在浏览器与浏览器之间，通过官方提供的SDK，我们可以很容易的实现本地应用间的音视频传输。在Android平台上，我们也非常容易的集成WebRTC框架，用非常简洁的代码就能实现强大、可靠的音视频传输功能。

接下的来的部分，我们会一起搭建Android平台的WebRTC demo，并实现两端之间的局域网传屏功能，同时还支持相互间消息数据的发送。

## 导入WebRTC官方aar

google官方已经提供了打包好的so与java层sdk代码，可方便的直接导入aar包。

```gradle
implementation 'org.webrtc:google-webrtc:1.0.32006'
```

如果对api部分或者so底层有修改，想自行发布编译好的aar咋办？在官方源码`src/tools_webrtc/android/`中`build_aar.py`与`release_aar.py`中有相关生成本地aar与发布aar到maven仓库的脚本。

当然，你也可自行编译so与引入java层sdk代码到项目中。但生成aar的sdk源码并不是放在一个位置，而是分散在WebRTC各个模块中，我们可以通过源码中`src/sdk/android/BUILD.gn`文件`dist_jar("libwebrtc")`任务查看相关代码依赖。


## 初始化`PeerConnectionFactory`

在初次使用`PeerConnectionFactory`之前，必须调用静态方法`initialize()`对其进行全局的初始化与资源加载。其中传参`InitializationOptions`通过内部Builder进行初始化，可对`LibraryLoader`、`Tracer`、`Logger`等进行设置，一般推荐放在`Application`中进行调用。

```java
PeerConnectionFactory.initialize(PeerConnectionFactory
        .InitializationOptions
        .builder(this)
        .setEnableInternalTracer(true)
        .createInitializationOptions());
```

## 创建`PeerConnectionFactory`对象

完成全局初始化后，我们就可以创建`PeerConnectionFactory`实例了。这个工厂类非常重要，在后续创建连接及音视频采集/编解码中，需要其为我们生成各种重要的组件。如：`PeerConnection`、`VideoSource`、`VideoTrack`等...采用Builder模式对其进行初始化，可方便对其进行编解码器的设置。

```java
final VideoEncoderFactory encoderFactory = new DefaultVideoEncoderFactory(mEglBase.getEglBaseContext(), true, true);
final VideoDecoderFactory decoderFactory = new DefaultVideoDecoderFactory(mEglBase.getEglBaseContext());
mPeerConnectionFactory = PeerConnectionFactory.builder()
        .setVideoEncoderFactory(encoderFactory)
        .setVideoDecoderFactory(decoderFactory)
        .createPeerConnectionFactory();
```

这里我们采用默认的`DefaultVideoEncoderFactory`和`DefaultVideoDecoderFactory`即可，可以简单看下内部的实现，以Decoder为例，其实内部同时支持软解与硬解，首选硬解，若硬解不支持则回退到软解：
```java
public class DefaultVideoDecoderFactory implements VideoDecoderFactory {
  ......
  @Override
  public @Nullable
  VideoDecoder createDecoder(VideoCodecInfo codecType) {
    VideoDecoder softwareDecoder = softwareVideoDecoderFactory.createDecoder(codecType);
    final VideoDecoder hardwareDecoder = hardwareVideoDecoderFactory.createDecoder(codecType);
    if (softwareDecoder == null && platformSoftwareVideoDecoderFactory != null) {
      softwareDecoder = platformSoftwareVideoDecoderFactory.createDecoder(codecType);
    }
    if (hardwareDecoder != null && softwareDecoder != null) {
      // Both hardware and software supported, wrap it in a software fallback
      return new VideoDecoderFallback(
          /* fallback= */ softwareDecoder, /* primary= */ hardwareDecoder);
    }
    return hardwareDecoder != null ? hardwareDecoder : softwareDecoder;
  }
  ......
}
```

## 通过Factory创建`PeerConnection`对象

生成了Factory后，我们就可以开始创建`PeerConnection`对象了。顾名思义，这个类代表点对点之间的连接，可以从远端获取音视频流等数据。在创建之前可以通过`RTCConfiguration`对连接进行详细的配置，最后通过`createPeerConnection()`方法完成创建。

```java
PeerConnection.RTCConfiguration rtcConfig =
        new PeerConnection.RTCConfiguration(iceServers);
// TCP candidates are only useful when connecting to a server that supports
// ICE-TCP.
rtcConfig.tcpCandidatePolicy = PeerConnection.TcpCandidatePolicy.DISABLED;
rtcConfig.bundlePolicy = PeerConnection.BundlePolicy.MAXBUNDLE;
rtcConfig.rtcpMuxPolicy = PeerConnection.RtcpMuxPolicy.REQUIRE;
rtcConfig.continualGatheringPolicy = PeerConnection.ContinualGatheringPolicy.GATHER_CONTINUALLY;
// Use ECDSA encryption.
rtcConfig.keyType = PeerConnection.KeyType.ECDSA;
// Enable DTLS for normal calls and disable for loopback calls.
rtcConfig.enableDtlsSrtp = true;
rtcConfig.sdpSemantics = PeerConnection.SdpSemantics.UNIFIED_PLAN;

mPeerConnection = peerConnectionFactory.createPeerConnection(rtcConfig, this);
```

## 创建音视频数据源

除此之外，我们还可以使用Factory创建音视频的数据源。通过`createVideoSource()`与`createAudioSource()`方法即可快速创建数据源。但这里的数据源仅是抽象的表示，那具体的数据从哪里来呢？

对于音频来说，在创建`AudioSource`时，就开始从音频设备捕获数据了。对于视频流，WebRTC中定义了`VideoCapturer`抽象接口，并提供了3种实现：`ScreenCapturerAndroid`、`CameraCapturer`和`FileVideoCapturer`，分别为从录屏、摄像头及文件中获取视频流，调用`startCapture()`后将开始获取数据。

```java
// Create video source
SurfaceTextureHelper surfaceTextureHelper = SurfaceTextureHelper.create("CaptureThread", mEglBase.getEglBaseContext());
mVideoSource = mPeerConnectionFactory.createVideoSource(capturer.isScreencast());
capturer.initialize(surfaceTextureHelper, this, mVideoSource.getCapturerObserver());
capturer.startCapture(1920, 1080, 30);

// Create audio source
mAudioSource = mPeerConnectionFactauory.createAudioSource(new MediaConstraints());
```

`VideoCapturer`这里采用了观察者模式，当获取到视频流的时候，会通过传入的`CapturerObserver`进行回调，以完成与`VideoSource`的关联。
```java
public interface CapturerObserver {
  void onCapturerStarted(boolean success);
  void onCapturerStopped();
  void onFrameCaptured(VideoFrame frame);
}
```

最后通过`createVideoTrack()`与`createAudioTrack()`完成对Source的包装，对于视频轨`VideoTrack`，我们可以通过`addSink()`方法传入`SurfaceViewRenderer`以对视频流进行本地的渲染展示（类似于视频会议场景显示本地的视频流）

```java
// Create video track
VideoTrack videoTrack = mPeerConnectionFactory.createVideoTrack(VIDEO_TRACK_ID, mVideoSource);
videoTrack.setEnabled(true);
videoTrack.addSink(mLocalSurfaceView);

// Create audio track
AudioTrack audioTrack = mPeerConnectionFactory.createAudioTrack(AUDIO_TRACK_ID, mAudioSource);
audioTrack.setEnabled(true);
```

其中`SurfaceViewRenderer`为`VideoSink`接口的实现类，我们可以把`VideoSink`抽象的当做视频流的接收方，由它来决定视频流该如何处理。`SurfaceViewRenderer`在接收到`onFrame()`回调后，内部会调用OpenGL进行渲染。

```java
public interface VideoSink {
  @CalledByNative
  void onFrame(VideoFrame frame);
}
```

## 添加`MediaStreamTrack`

创建好`VideoTrack`和`AudioTrack`后，我们就可以通过`PeerConnection`把音视频轨添加进去了。这样WebRTC 才能帮我们生成包含相应媒体信息的SDP，以便于后面做媒体能力协商使用。要注意`addTrack()`必须早于后续的商阶段，否则另一端无法收到相关的音视频数据。

```java
mPeerConnection.addTrack(videoTrack, mediaStreamLabels);
mPeerConnection.addTrack(audioTrack, mediaStreamLabels);
```

## 建立信令服务器

在建立连接之前，我们必须通过信令服务器来交换SDP信息。简单起见，我们的demo采用局域网的传输方式，参考官方demo，直接采用java Socket实现（也可选择`Netty`或者`socket.io`等第三方框架），详见`TCPChannelClient`。代码较简单，判断传入的IP为local地址，则作为服务器，否则作为客户端，并向上层提供发送数据的接口。

```java
  public TCPChannelClient(
          ExecutorService executor, TCPChannelEvents eventListener, String ip, int port) {
    this.executor = executor;
    executorThreadCheck = new ThreadUtils.ThreadChecker();
    executorThreadCheck.detachThread();
    this.eventListener = eventListener;

    InetAddress address;
    try {
      address = InetAddress.getByName(ip);
    } catch (UnknownHostException e) {
      reportError("Invalid IP address.");
      return;
    }

    if (address.isAnyLocalAddress()) {
      socket = new TCPSocketServer(address, port);
    } else {
      socket = new TCPSocketClient(address, port);
    }

    socket.start();
  }
```

## 进行媒体协商

跟之前分析的Miracast RTSP协议类似，在进行音视频流传输之前，需要进行能力协商。实际就是你的设备所支持的音视频编解码器、使用的传输协议、SSRC等信息...通过信令服务器透传给对方。如果双方都支持，那么就算协商成功了。

- Offer：呼叫方发送的SDP消息称为Offer
- Answer：被呼叫方发送的SDP消息称为Answer

双方协商的整个过程如下图所示：

![](offer-answer.png)

这里，我们以连接上的Client端为呼叫方先发起Offer请求，通过`createOffer()`创建一个Offer SDP。创建成功后，会在`SdpObserver`中收到`onCreateSuccess`回调，此时调用`setLocalDescription()`方法将该Offer保存到本地Local域，然后将Offer发送给对方。

```java
public class PeerConnectionWrapper implements PeerConnection.Observer, SdpObserver {
    ......
    public void createOffer() {
        mIsInitiator = true;
        mPeerConnection.createOffer(this, mSdpMediaConstraints);
    }

    public void createAnswer() {
        mIsInitiator = false;
        mPeerConnection.createAnswer(this, mSdpMediaConstraints);
    }
    ......
    @Override
    public void onCreateSuccess(SessionDescription sessionDescription) {
        Log.d(TAG, "onCreateSuccess: " + sessionDescription.description);
        if (mIsInitiator) {
            mRTCClient.sendOfferSdp(sessionDescription);
        } else {
            mRTCClient.sendAnswerSdp(sessionDescription);
        }
        mPeerConnection.setLocalDescription(this, sessionDescription);
    }
    ......
}
```

被呼叫方接收到Offer后，通过`setRemoteDescription()`方法将Offer保存到它的Remote域，并通过`createAnswer()`创建Answer SDP，创建成功后同样调用`setLocalDescription()`方法将Answer消息保存到本地的Local域，然后回复给呼叫方。

最后，呼叫方将收到Answer消息，并通过`setRemoteDescription()`方法，将Answer保存到它的Remote域。至此，整个媒体协商的过程结束。

```java
mRTCClient = new DirectRTCClient(new AppRTCClient.SignalingCallback() {
    ......
    @Override
    public void onRemoteAnswer(SessionDescription sdp) {
        mPeerConnectionWrapper.getPeerConnection().setRemoteDescription(mPeerConnectionWrapper, sdp);
    }

    @Override
    public void onRemoteOffer(SessionDescription sdp) {
        mPeerConnectionWrapper.getPeerConnection().setRemoteDescription(mPeerConnectionWrapper, sdp);
        mPeerConnectionWrapper.createAnswer();
    }
});
```

## 建立点对点连接

在媒体协商结束后，我们的点对点连接并没有真正的建立。此时`createPeerConnection()`中传入的`PeerConnection.Observer`会回调`onIceCandidate()`方法并提供`IceCandidate`对象，这个时候我们把他组装为candidate的SDP信令发送到信令服务器，透传给另外一端。

```java
public class PeerConnectionWrapper implements PeerConnection.Observer, SdpObserver {
    ......
    @Override
    public void onIceCandidate(IceCandidate candidate) {
        Log.d(TAG, "onIceCandidate:");
        mRTCClient.sendLocalIceCandidate(candidate);
    }
}
```

远端在收到`IceCandidate`对象后进行重建，并通过`addIceCandidate()`方法将其添加进`PeerConnection`中。

```java
mRTCClient = new DirectRTCClient(new AppRTCClient.SignalingCallback() {
    ......
    @Override
    public void onRemoteIceCandidate(IceCandidate candidate) {
        mPeerConnectionWrapper.getPeerConnection().addIceCandidate(candidate);
    }
}
```

接下来双方获取到彼此的Candidate之后，WebRTC就开始尝试进行连接了。优先级：host > srflx > relay，host 类型之间的连通性检测就是内网之间的连通性检测，上述场景中我们的呼叫双方都在同一个局域网中，因此将以host的方式进行连接。


## 展示远端视频流

当点对点连接建立起来后，我们就可以开始获取音视频流数据了。之前在`createPeerConnection()`中传入的`PeerConnection.Observer`会回调`onAddStream()`方法（注意此方法会在收到远端SDP并调用`setRemoteDescription()`后，就会回调了，不用等连接真正建立，与`onAddTrack()`一致），并提供`MediaStream`对象，其中包含远端的音视频轨`AudioTracks`与`VideoTracks`。前面我们添加了一条录屏的视频轨，因此直接获取第一条`VideoTrack`对象即可，然后跟之前一样通过`addSink()`即可与`SurfaceViewRenderer`的绑定，从而渲染出视频流。

```java
@Override
public void onAddStream(MediaStream mediaStream) {
    Log.d(TAG, "onAddStream audio tracks size:" + mediaStream.audioTracks.size() + " video" + mediaStream.videoTracks.size());
    if (mediaStream.videoTracks.size() >= 1) {
        // Assuming there is only one video track.
        VideoTrack remoteVideoTrack = mediaStream.videoTracks.get(0);
        remoteVideoTrack.setEnabled(true);
        remoteVideoTrack.addSink(mRemoteVideoSink);
    }
}
```

## 关于音频的录制及播放

在WebRTC中一般通过`JavaAudioDeviceModule`来实现音视频的录制及播放，底层采用`AudioRecord`来进行录制及`AudioTrack`来进行音频播放，通过Builder来构建实例。并在创建`PeerConnectionFactory`的时候通过`setAudioDeviceModule()`方法设置即可。

```java
private AudioDeviceModule createJavaAudioDevice() {
    ......
    return JavaAudioDeviceModule.builder(getApplicationContext())
            .setUseHardwareAcousticEchoCanceler(false)
            .setUseHardwareNoiseSuppressor(false)
            .setAudioRecordErrorCallback(audioRecordErrorCallback)
            .setAudioTrackErrorCallback(audioTrackErrorCallback)
            .setAudioRecordStateCallback(audioRecordStateCallback)
            .setAudioTrackStateCallback(audioTrackStateCallback)
            .createAudioDeviceModule();
}

private void initPeerConnection() {
    final AudioDeviceModule adm = createJavaAudioDevice();
    mPeerConnectionFactory = PeerConnectionFactory.builder()
            .setAudioDeviceModule(adm)
            ......
            .createPeerConnectionFactory();
    ......
    adm.release();
}
```

## 使用`DataChannel`发送消息

WebRTC的数据通道`DataChannel`专门用来传输音视频流外的任何数据，所以它的应用非常广泛，如实时文字聊天、文件传输等...其中`DataChannel`有两种创建方式，一种是默认的`In-band`协商方式，另一种是`Out-of-band`协商方式，根据`negotiated`字段来初始化。

### `In-band`协商

一端需调用`createDataChannel()`创建`DataChannel`对象，并设置`negotiated`为`false`（默认值）：

```java
DataChannel.Init init = new DataChannel.Init();
init.ordered = true;
init.negotiated = false;
mDataChannel = mPeerConnection.createDataChannel("dataChannel", init);
mDataChannel.registerObserver(new DataChannel.Observer() {
    ...
    @Override
    public void onMessage(DataChannel.Buffer buffer) {
        // Receive message from remote
        ......
    }
});
```

当媒体协商完成，并且连接建立好后，另一端则会通过`PeerConnection.Observer`的`onDataChannel()`回调获取到对应的数据通道，此时可通过参数`DataChannel`进行数据的回复：
```java
@Override
public void onDataChannel(final DataChannel dataChannel) {
    // Triggered when a remote peer opens a DataChannel
    dataChannel.registerObserver(new DataChannel.Observer() {
        ...
        @Override
        public void onMessage(DataChannel.Buffer buffer) {
            // Replay message to remote
            sendDataChannelMessage("Replay message from:" + dataChannel);
            ......
        }
    });
}
```

双方通过`DataChannel.send()`方法即可相互发送数据：
```java
public void sendDataChannelMessage(String message, DataChannel dataChannel) {
    byte[] msg = message.getBytes();
    DataChannel.Buffer buffer = new DataChannel.Buffer(
            ByteBuffer.wrap(msg), false);
    dataChannel.send(buffer);
}
```

在另一端`DataChannel.Observer()`的`onMessage()`回调中，我们可以获取到远端发送的数据：
```java
ByteBuffer data = buffer.data;
final byte[] bytes = new byte[data.capacity()];
data.get(bytes);
String strData = new String(bytes, Charset.forName("UTF-8"));
Log.d(TAG, "Got msg: " + strData + " over " + mDataChannel.label() + " id:" + mDataChannel.id());
```

### `Out-of-band`协商

两端都调用`createDataChannel()`方法创建`DataChannel`对象，并设置`negotiated`为`true`，再通过ID绑定来实现双方的数据通信。此方式的优点是双方发送数据时不用考虑时序问题，代码也更简洁一点，需注意绑定的ID必须一致：

```java
DataChannel.Init init = new DataChannel.Init();
init.ordered = true; // 消息的传递是否有序
init.negotiated = true; // 协商方式
init.id = 0; // 通道ID
// init.maxPacketLifeTime // 重传最大超时时间
// init.maxRetransmits  // 重传最大次数
mDataChannel = mPeerConnection.createDataChannel("dataChannel", init);
mDataChannel.registerObserver(new DataChannel.Observer() {
    ...
    @Override
    public void onMessage(DataChannel.Buffer buffer) {
        // Receive message from remote
        ......
    }
});
```

## 参考
[极客时间《从0打造音视频直播系统》](https://time.geekbang.org/column/intro/207)

[WebRTC入门教程(三) | Android 端如何使用 WebRTC](https://rtcdeveloper.com/t/topic/14040)
