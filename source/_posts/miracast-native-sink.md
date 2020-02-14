---
title: Miracast技术详解（四）：Sink源码解析
date: 2020-02-07 09:34:16
categories: Miracast
tags: [Android, Miracast, Sink, RTFSC]
---

## Sink源码概述

Miracast Sink端源码最早出现在`Android 4.2.2`上，通过`googlesource`可以很方便的查看：
https://android.googlesource.com/platform/frameworks/av/+/android-4.2.2_r1.2/media/libstagefright/wifi-display/sink/

但是在`Android 4.3`以后，Google却移除掉了这部分源码，详细的commit记录在：
https://android.googlesource.com/platform/frameworks/av/+/c4bd06130e4c3068ab58a0be88a4f765c2267563

```
Remove all traces of wifi display sink implementation and supporting code.

Change-Id: I64b681b7e3df1ef0dd80c0d261cacae293d5e684
related-to-bug: 8698812
```

虽然移除了Sink端代码，但是Source端源码是还在的，我们可以通过Android手机的投射功能实现Miracast投屏发送端。

## 导入源码
这里推荐使用Android Studio进行源码查看，为了方便使用IDE的代码提示及类/方法跳转等相关功能，我们需要搭建好源码环境。首先新建一个`Native Project`，然后把整个`libstagefright`相关的源码拷贝到cpp目录中，最好把相关的`include`头文件也一起导入（因为涉及到很多依赖），然后在`CMakeLists.txt`中添加这部分源码。重新sync一次，这样就能引用到相关的类与头文件，并且支持代码提示，提高我们查看源码的效率。
```
include_directories(include)

file(GLOB_RECURSE MIRACASTSRC
        "media/libstagefright/*.h"
        "media/libstagefright/*.cpp"
        )

add_library( # Sets the name of the library.
        native-lib

        # Sets the library as a shared library.
        SHARED

        # Provides a relative path to your source file(s).
        native-lib.cpp ${MIRACASTSRC})
```

![](as-sink.jpg)

## RTFSC

Sink端源码主要的核心类就这3个：`WifiDisplaySink.cpp`、`RTPSink.cpp`、`TunnelRenderer.cpp`。我们可以从`wfd.cpp`这个可执行程序的`main()`方法看起，看Sink端是如何进行初始化的。

最重要的几行初始化代码在`main()`函数的最后，几个需要关注的点：
- 创建了`ANetworkSession`对象，并启动了内部的`NetworkThread`，此类主要用来管理通信相关的TCP和UDP连接。
- 创建并启动了`ALooper`，并将`WifiDisplaySink`通过`registerHandler()`方法关联起来。其中`WifiDisplaySink`、`RTPSink`、`TunnelRenderer`都继承了`AHandlr`，都实现了`onMessageReceived()`，可以用来处理异步消息。
- 整个程序采用了`ALooper`，`AHandler`，`AMessage`这几个类配合完成的异步消息机制，它是`Android Native`层实现的一个异步消息机制，跟应用层的`Handler`那一套有点类似。

```c++
sp<ANetworkSession> session = new ANetworkSession;
session->start();

sp<ALooper> looper = new ALooper;

sp<WifiDisplaySink> sink = new WifiDisplaySink(session);
looper->registerHandler(sink);

if (connectToPort >= 0) {
    sink->start(connectToHost.c_str(), connectToPort);
} else {
    sink->start(uri.c_str());
}

looper->start(true /* runOnCallingThread */);
```

最后通过关键的`sink->start()`方法启动`WifiDisplaySink`，我们来看下里面的操作，以ip和端口这个`start()`方法为例，post了一个类型为`kWhatStart`的`AMessage`对象：
```c++
void WifiDisplaySink::start(const char *sourceHost, int32_t sourcePort) {
    sp<AMessage> msg = new AMessage(kWhatStart, id());
    msg->setString("sourceHost", sourceHost);
    msg->setInt32("sourcePort", sourcePort);
    msg->post();
}
```

### RTSP通讯

前面我们说到了`WifiDisplaySink`继承了`AHandler`，因此消息最终会回调到`onMessageReceived()`方法中去处理。这里通过`createRTSPClient()`方法创建了RTSP的TCP连接，并传入了一个`AMessage`对象，以用作后面的连接状态与数据的异步通知：
```c++
void WifiDisplaySink::onMessageReceived(const sp<AMessage> &msg) {
    switch (msg->what()) {
        case kWhatStart:
        {
            ...
            sp<AMessage> notify = new AMessage(kWhatRTSPNotify, id());

            status_t err = mNetSession->createRTSPClient(
                    mRTSPHost.c_str(), sourcePort, notify, &mSessionID);
            CHECK_EQ(err, (status_t)OK);

            mState = CONNECTING;
            break;
        }
        ...
    }
}
```

当RTSP连接失败或成功后，会通过`kWhatError`与`kWhatConnected`进行通知回调。若连接成功建立，则后续会通过`kWhatData`收到RTSP相关的数据包。这个时候就正式开始RTSP协商与会话建立，涉及到RTSP的`M1-M7`指令处理。具体的Rqeuest及Response流程可以看前面的RTSP协议分析文章，这里不详细展开。
```c++
void WifiDisplaySink::onMessageReceived(const sp<AMessage> &msg) {
    switch (msg->what()) {
        ...
        case kWhatRTSPNotify:
        {
            int32_t reason;
            CHECK(msg->findInt32("reason", &reason));

            switch (reason) {
                case ANetworkSession::kWhatError:
                {
                    // 处理网络异常状态，忽略
                }

                case ANetworkSession::kWhatConnected:
                {
                    // 网络连接成功回调，并设置状态为已连接
                    ALOGI("We're now connected.");
                    mState = CONNECTED;
                    ...
                }

                case ANetworkSession::kWhatData:
                {
                    // 处理接收到的RTSP数据包
                    onReceiveClientData(msg);
                    break;
                }
                ...
            }
            break;
        }
        ...
    }
}
```

首先判断消息类型，看看是Request还是Response，这里的做法有点暴力，判断`RTSP/`开头则认为是Response。详细解析看注释：
```c++
void WifiDisplaySink::onReceiveClientData(const sp<AMessage> &msg) {
    ...
    if (method.startsWith("RTSP/")) {
        // This is a response.

        ResponseID id;
        id.mSessionID = sessionID;
        id.mCSeq = cseq;
        // mResponseHandlers是key-value结构，用来存储对应请求的ResponseHandler
        // 根据RTSP的交互流程，在调用sendM2, sendSetup(M6), sendPlay(M7)的时候由Sink主动发起Request
        // 这个时候会调用mResponseHandlers.add()添加对应请求的ResponseHandler
        ssize_t index = mResponseHandlers.indexOfKey(id);

        if (index < 0) {
            ALOGW("Received unsolicited server response, cseq %d", cseq);
            return;
        }
        // 取出对应ResponseID的HandleRTSPResponseFunc，并从映射表中移除
        HandleRTSPResponseFunc func = mResponseHandlers.valueAt(index);
        mResponseHandlers.removeItemsAt(index);
        // 调用对应的HandleRTSPResponseFunc进行回调，这个func我们可以看做一个callback
        status_t err = (this->*func)(sessionID, data);
        CHECK_EQ(err, (status_t)OK);
    } else {
        AString version;
        data->getRequestField(2, &version);
        // RTSP version合法性判断
        if (!(version == AString("RTSP/1.0"))) {
            sendErrorResponse(sessionID, "505 RTSP Version not supported", cseq);
            return;
        }
        if (method == "OPTIONS") {
            // 对应Source端发起的OPTIONS M1请求
            onOptionsRequest(sessionID, cseq, data);
        } else if (method == "GET_PARAMETER") {
            // 对应GET_PARAMETER M3请求
            onGetParameterRequest(sessionID, cseq, data);
        } else if (method == "SET_PARAMETER") {
            // 对应SET_PARAMETER M4与M5请求
            onSetParameterRequest(sessionID, cseq, data);
        } else {
            sendErrorResponse(sessionID, "405 Method Not Allowed", cseq);
        }
    }
}
```

针对Request部分，我们继续根据RTSP流程进行细化，`onOptionsRequest()`主要处理Source端M1请求，并且响应完后马上执行`sendM2()`方法，以确认Source端所支持的RTSP方法请求。
```c++
void WifiDisplaySink::onOptionsRequest(
        int32_t sessionID,
        int32_t cseq,
        const sp<ParsedMessage> &data) {
    AString response = "RTSP/1.0 200 OK\r\n";
    AppendCommonResponse(&response, cseq);
    response.append("Public: org.wfa.wfd1.0, GET_PARAMETER, SET_PARAMETER\r\n");
    response.append("\r\n");

    status_t err = mNetSession->sendRequest(sessionID, response.c_str());
    CHECK_EQ(err, (status_t)OK);

    err = sendM2(sessionID);
    CHECK_EQ(err, (status_t)OK);
}
```

`onGetParameterRequest()`则是处理Source端M3请求，并返回自持自身支持的属性及能力，比较重要的几个属性：RTP端口号（传输流媒体用）、所支持的audio及video编解码格式等...
```c++
void WifiDisplaySink::onGetParameterRequest(
        int32_t sessionID,
        int32_t cseq,
        const sp<ParsedMessage> &data) {
    AString body =
        "wfd_video_formats: xxx\r\n"
        "wfd_audio_codecs: xxx\r\n"
        "wfd_client_rtp_ports: RTP/AVP/UDP;unicast xxx 0 mode=play\r\n";

    AString response = "RTSP/1.0 200 OK\r\n";
    AppendCommonResponse(&response, cseq);
    response.append("Content-Type: text/parameters\r\n");
    response.append(StringPrintf("Content-Length: %d\r\n", body.size()));
    response.append("\r\n");
    response.append(body);

    status_t err = mNetSession->sendRequest(sessionID, response.c_str());
    CHECK_EQ(err, (status_t)OK);
}
```

`onSetParameterRequest()`则是处理Source端M4与M5请求，对于M4请求，直接进行常规的Response即可。对于M5，除了Response之外，因为请求中带有`wfd_trigger_method: SETUP`，会触发Sink端向Source端发送SETUP请求。
```c++
void WifiDisplaySink::onSetParameterRequest(
        int32_t sessionID,
        int32_t cseq,
        const sp<ParsedMessage> &data) {
    const char *content = data->getContent();

    if (strstr(content, "wfd_trigger_method: SETUP\r\n") != NULL) {
        status_t err =
            sendSetup(
                    sessionID,
                    "rtsp://x.x.x.x:x/wfd1.0/streamid=0");

        CHECK_EQ(err, (status_t)OK);
    }

    AString response = "RTSP/1.0 200 OK\r\n";
    AppendCommonResponse(&response, cseq);
    response.append("\r\n");

    status_t err = mNetSession->sendRequest(sessionID, response.c_str());
    CHECK_EQ(err, (status_t)OK);
}
```

在`sendSetup()`方法中，有两个比较重要的点。一是我们初始化了`RTPSink`，主要用于后续建立UDP连接与处理RTP包。二是在发送完`Setup M6`请求后，注册了`onReceiveSetupResponse()`回调。
```c++
status_t WifiDisplaySink::sendSetup(int32_t sessionID, const char *uri) {
    mRTPSink = new RTPSink(mNetSession, mSurfaceTex);
    looper()->registerHandler(mRTPSink);

    status_t err = mRTPSink->init(sUseTCPInterleaving);

    if (err != OK) {
        looper()->unregisterHandler(mRTPSink->id());
        mRTPSink.clear();
        return err;
    }
    ...
    err = mNetSession->sendRequest(sessionID, request.c_str(), request.size());
    ...
    registerResponseHandler(
            sessionID, mNextCSeq, &WifiDisplaySink::onReceiveSetupResponse);
    ...
    return OK;
}
```

在`onReceiveSetupResponse()`方法的最后我们完成了RTSP中的最后一步，发送`PLAY M7`请求，告诉发送端可以开始发送流媒体数据了。这个时候Source端会按照Sink指定的UDP端口（SETUP指令中指定）发送RTP数据包，包含音视频数据。
```c++
status_t WifiDisplaySink::sendPlay(int32_t sessionID, const char *uri) {
    AString request = StringPrintf("PLAY %s RTSP/1.0\r\n", uri);

    AppendCommonResponse(&request, mNextCSeq);

    request.append(StringPrintf("Session: %s\r\n", mPlaybackSessionID.c_str()));
    request.append("\r\n");

    status_t err =
        mNetSession->sendRequest(sessionID, request.c_str(), request.size());
    ...
    return OK;
}
```

经过以上代码，RTSP的协商与会话建立就已经完成了，并且能在指定的UDP端口中收到音视频RTP数据包。看完这部分Native代码，我们完全可以在应用层用Socket或者Netty这样的第三方网络框架实现。下面我们继续分析`RTPSink`中是如何处理RTP的数据包的。

### RTP通讯

前面我们提到，在`sendSetup()`方法中，我们初始化了`RTPSink`，并且在`onReceiveSetupResponse()`回调中调用`configureTransport()`方法，根据对应的RTP与RTCP端口，调用`mRTPSink->connect()`方法建立UDP连接。
```c++
status_t WifiDisplaySink::configureTransport(const sp<ParsedMessage> &msg) {
    ...
    int rtpPort, rtcpPort;
    if (sscanf(serverPortStr.c_str(), "%d-%d", &rtpPort, &rtcpPort) != 2
            || rtpPort <= 0 || rtpPort > 65535
            || rtcpPort <=0 || rtcpPort > 65535
            || rtcpPort != rtpPort + 1) {
        ALOGE("Invalid server_port description '%s'.",
                serverPortStr.c_str());

        return ERROR_MALFORMED;
    }

    if (rtpPort & 1) {
        ALOGW("Server picked an odd numbered RTP port.");
    }

    return mRTPSink->connect(sourceHost.c_str(), rtpPort, rtcpPort);
}
```

在`connect()`方法中，通过`ANetworkSession`完成对应RTP与RTCP连接的connect操作。其中RTP与RTCP的Session在init()方法调用的时候已经创建完毕，这里不再展示。
```c++
status_t RTPSink::connect(
        const char *host, int32_t remoteRtpPort, int32_t remoteRtcpPort) {
    ALOGI("connecting RTP/RTCP sockets to %s:{%d,%d}",
          host, remoteRtpPort, remoteRtcpPort);

    status_t err =
        mNetSession->connectUDPSession(mRTPSessionID, host, remoteRtpPort);

    if (err != OK) {
        return err;
    }

    err = mNetSession->connectUDPSession(mRTCPSessionID, host, remoteRtcpPort);

    if (err != OK) {
        return err;
    }
    ...
}
```

前面我们提到了`RTPSink`继承了`AHandler`，因此创建相关`UDPSession`传入的`kWhatRTPNotify`与`kWhatRTCPNotify`消息最终会回调到`onMessageReceived()`方法中去处理。
```c++
void RTPSink::onMessageReceived(const sp<AMessage> &msg) {
    switch (msg->what()) {
        case kWhatRTPNotify:
        case kWhatRTCPNotify:
        {
            int32_t reason;
            CHECK(msg->findInt32("reason", &reason));

            switch (reason) {
                case ANetworkSession::kWhatError:
                {
                    // 处理网络异常状态，忽略
                }

                case ANetworkSession::kWhatDatagram:
                {
                    int32_t sessionID;
                    CHECK(msg->findInt32("sessionID", &sessionID));

                    sp<ABuffer> data;
                    // 取出ABuffer中的RTP或者RTCP数据包
                    CHECK(msg->findBuffer("data", &data));

                    status_t err;
                    // 根据消息类型分别进行RTP与RTCP包的解析
                    if (msg->what() == kWhatRTPNotify) {
                        err = parseRTP(data);
                    } else {
                        err = parseRTCP(data);
                    }
                    break;
                }
                ...
            }
            break;
        }
        ...
    }
}
```

在前面的文章中我们也提到，经过多台手机的抓包测试，发现抓到的UDP数据中只包含了RTP数据包，而没有发现RTCP数据包。因此我们这里的分析先不考虑RTCP数据包的处理，直接看RTP包的解析即可。详细解析看注释：
```c++
status_t RTPSink::parseRTP(const sp<ABuffer> &buffer) {
    size_t size = buffer->size();
    if (size < 12) {
        // Too short to be a valid RTP header.
        // RTP的固定包头至少12字节，包含CSRC的时候会超过12字节
        return ERROR_MALFORMED;
    }

    const uint8_t *data = buffer->data();

    // 按照以下RTP固定包头进行解析
    //  0                   1                   2                   3
    //  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    // +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    // |V=2|P|X|  CC   |M|     PT      |       sequence number         |
    // +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    // |                           timestamp                           |
    // +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    // |           synchronization source (SSRC) identifier            |
    // +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
    // |            contributing source (CSRC) identifiers             |
    // |                             ....                              |
    // +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    ...

    // 将解析出来的ssrc、timestamp、payload type、marker等属性设置进buffer的meta域
    sp<AMessage> meta = buffer->meta();
    meta->setInt32("ssrc", srcId);
    meta->setInt32("rtp-time", rtpTime);
    meta->setInt32("PT", data[1] & 0x7f);
    meta->setInt32("M", data[1] >> 7);
    // 包头后剩下的字节为MPEG2-TS Payload
    buffer->setRange(payloadOffset, size - payloadOffset);
    // 根据SSRC获取对应的Source，在一个RTP传输周期内，相同的同步源将具有相同的SSRC
    ssize_t index = mSources.indexOfKey(srcId);
    // 若mSources中查找不到，则创建新的Source并加入到映射表中
    if (index < 0) {
        if (mRenderer == NULL) {
            sp<AMessage> notifyLost = new AMessage(kWhatPacketLost, id());
            notifyLost->setInt32("ssrc", srcId);
            // 创建TunnelRenderer，为后续的音视频渲染做准备
            mRenderer = new TunnelRenderer(notifyLost, mSurfaceTex);
            looper()->registerHandler(mRenderer);
        }

        sp<AMessage> queueBufferMsg =
            new AMessage(TunnelRenderer::kWhatQueueBuffer, mRenderer->id());

        sp<Source> source = new Source(seqNo, buffer, queueBufferMsg);
        mSources.add(srcId, source);
    } else {
        // 若已经创建好了Source，直接将buffer入队
        mSources.valueAt(index)->updateSeq(seqNo, buffer);
    }

    return OK;
}
```

在`updateSeq()`方法中，会对sequence包序列号进行一些处理，如：丢包间隙过大、重复包、64K周期循环处理等，最终通过`queuePacket()`方法将MPEG2-TS数据包入队。其中`mQueueBufferMsg`为`TunnelRenderer::kWhatQueueBuffer`类型的消息，最终会在`TunnelRenderer`中进行处理。
```c++
void RTPSink::Source::queuePacket(const sp<ABuffer> &buffer) {
    sp<AMessage> msg = mQueueBufferMsg->dup();
    msg->setBuffer("buffer", buffer);
    msg->post();
}
```

### 播放阶段

数据包来到了`TunnelRenderer`，它代表一个呈现通道，会接收`queuePacket()`解包出来的MPEG2-TS音视频流，然后进行播放。其中`TunnelRenderer`继承了`AHandler`，因此消息最终会回调到`onMessageReceived()`方法中去处理。里面的处理也比较简单，主要是buffer入队操作及环境初始化。
```c++
void TunnelRenderer::onMessageReceived(const sp<AMessage> &msg) {
    switch (msg->what()) {
        case kWhatQueueBuffer:
        {
            sp<ABuffer> buffer;
            CHECK(msg->findBuffer("buffer", &buffer));
            // 将buffer数据入队
            queueBuffer(buffer);
            // 判断StreamSource为空则初始化MediaPlayerService等渲染环境
            if (mStreamSource == NULL) {
                if (mTotalBytesQueued > 0ll) {
                    initPlayer();
                } else {
                    ALOGI("Have %lld bytes queued...", mTotalBytesQueued);
                }
            } else {
                // 主要是将TS数据出队，并解析成音视频裸流进行渲染
                mStreamSource->doSomeWork();
            }
            break;
        }
        ...
    }
}
```

其中`queueBuffer()`操作主要将TS包添加到`mPackets`列表中，其中还对包进行了一些去重与重排序，保证包序列递增。而`initPlayer()`则对播放环境进行了初始化。详细注释如下：
```c++
void TunnelRenderer::initPlayer() {
    // 若mSurfaceTex为空（初始化WifiDisplaySink的时候没有传递mSurfaceTex），则由TunnelRenderer自行创建
    // 这里也意味着想要实现自己的Sink端，我们可以在应用层创建好SurfaceTexture，并一步步传递下来
    if (mSurfaceTex == NULL) {
        mComposerClient = new SurfaceComposerClient;
        CHECK_EQ(mComposerClient->initCheck(), (status_t)OK);

        DisplayInfo info;
        SurfaceComposerClient::getDisplayInfo(0, &info);
        ssize_t displayWidth = info.w;
        ssize_t displayHeight = info.h;
        // 创建SurfaceControl，并从中获取Surface实例
        mSurfaceControl =
            mComposerClient->createSurface(
                    String8("A Surface"),
                    displayWidth,
                    displayHeight,
                    PIXEL_FORMAT_RGB_565,
                    0);
        ...
        mSurface = mSurfaceControl->getSurface();
        CHECK(mSurface != NULL);
    }
    ...
    // 通过系统MediaPlayerService创建MediaPlayer实例
    mPlayer = service->create(getpid(), mPlayerClient, 0);
    CHECK(mPlayer != NULL);
    // 设置DataSource数据流
    CHECK_EQ(mPlayer->setDataSource(mStreamSource), (status_t)OK);
    // 对MediaPlayer设置SurfaceTexture，这样就能在对应的Surface上播放视频了
    mPlayer->setVideoSurfaceTexture(
            mSurfaceTex != NULL ? mSurfaceTex : mSurface->getSurfaceTexture());
    // 启动MediaPlayer，开始播放
    mPlayer->start();
}
```

TS包入队之后，会通过`mStreamSource->doSomeWork()`方法将TS包出队，并解析成音视频裸流进行渲染。详细解释如下注释：
```c++
void TunnelRenderer::StreamSource::doSomeWork() {
    Mutex::Autolock autoLock(mLock);

    while (!mIndicesAvailable.empty()) {
        // 将TS包出队
        sp<ABuffer> srcBuffer = mOwner->dequeueBuffer();
        if (srcBuffer == NULL) {
            break;
        }
        ...
        ALOGV("dequeue TS packet of size %d", srcBuffer->size());
        // 获取可用buffer的index
        size_t index = *mIndicesAvailable.begin();
        mIndicesAvailable.erase(mIndicesAvailable.begin());
        // 获取buffer对应的IMemory对象
        sp<IMemory> mem = mBuffers.itemAt(index);
        CHECK_LE(srcBuffer->size(), mem->size());
        // 检查Buffer大小，必须是188整数倍，因为单个TS包大小为188B
        CHECK_EQ((srcBuffer->size() % 188), 0u);
        // 通过内存拷贝的形式，将buffer发送到MediaPlayer进行解码与播放
        memcpy(mem->pointer(), srcBuffer->data(), srcBuffer->size());
        mListener->queueBuffer(index, srcBuffer->size());
    }
}
```

### MPEG2-TS解析

在`Native Sink`源码中，最终会通过`ATSParser.cpp`对TS包进行解析，拿出最终的音视频裸流。首先我们来看下调用的入口，在` MPEG2TSExtractor::feedMore()`方法中，会把数据包切割成一个个188B的标准TS包，并通过`feedTSPacket()`送入`ATSParser`开始解析。
```c++
status_t MPEG2TSExtractor::feedMore() {
    Mutex::Autolock autoLock(mLock);

    uint8_t packet[kTSPacketSize];
    ssize_t n = mDataSource->readAt(mOffset, packet, kTSPacketSize);

    if (n < (ssize_t)kTSPacketSize) {
        return (n < 0) ? (status_t)n : ERROR_END_OF_STREAM;
    }

    mOffset += n;
    return mParser->feedTSPacket(packet, kTSPacketSize);
}

status_t ATSParser::feedTSPacket(const void *data, size_t size) {
    CHECK_EQ(size, kTSPacketSize);

    ABitReader br((const uint8_t *)data, kTSPacketSize);
    return parseTS(&br);
}
```

开始解析TS包（对TS包格式不熟悉的，可以查看之前MPEG2-TS流解析文章），详细解析过程如下注释：
```c++
status_t ATSParser::parseTS(ABitReader *br) {
    ALOGV("---");
    // 检查TS包头的8位同步字节，固定为0x47
    unsigned sync_byte = br->getBits(8);
    CHECK_EQ(sync_byte, 0x47u);

    MY_LOGV("transport_error_indicator = %u", br->getBits(1));
    // 获取payload_unit_start_indicator，标记是否为一帧的起始
    unsigned payload_unit_start_indicator = br->getBits(1);
    ALOGV("payload_unit_start_indicator = %u", payload_unit_start_indicator);

    MY_LOGV("transport_priority = %u", br->getBits(1));
    // 获取TS包的PID
    unsigned PID = br->getBits(13);
    ALOGV("PID = 0x%04x", PID);

    MY_LOGV("transport_scrambling_control = %u", br->getBits(2));
    // 获取适配域
    unsigned adaptation_field_control = br->getBits(2);
    ALOGV("adaptation_field_control = %u", adaptation_field_control);

    unsigned continuity_counter = br->getBits(4);
    ALOGV("PID = 0x%04x, continuity_counter = %u", PID, continuity_counter);
    // 判断适配域，'10'表示仅有适配域，'11'表示适配域和Payload都存在
    if (adaptation_field_control == 2 || adaptation_field_control == 3) {
        // 开始解析适配域
        parseAdaptationField(br, PID);
    }

    status_t err = OK;
    // 判断适配域，'01'表示仅有Payload，'11'表示适配域和Payload都存在
    if (adaptation_field_control == 1 || adaptation_field_control == 3) {
        // 开始根据PID解析Payload
        err = parsePID(
                br, PID, continuity_counter, payload_unit_start_indicator);
    }

    ++mNumTSPacketsParsed;

    return err;
}
```

其中对适配域的解析`parseAdaptationField()`中比较关键的就是对PCR时钟的解析：
```c++
void ATSParser::parseAdaptationField(ABitReader *br, unsigned PID) {
    unsigned adaptation_field_length = br->getBits(8);

    if (adaptation_field_length > 0) {
        ...
        // 解析PCR flag，判断适配域头后是否有PCR时钟
        unsigned PCR_flag = br->getBits(1);
        ...
        if (PCR_flag) {
            br->skipBits(4);
            // 解析33比特的低精度部分
            uint64_t PCR_base = br->getBits(32);
            PCR_base = (PCR_base << 1) | br->getBits(1);

            br->skipBits(6);
            // 解析9比特的高精度部分
            unsigned PCR_ext = br->getBits(9);
            ...
            // 计算出最终的PCR时钟
            uint64_t PCR = PCR_base * 300 + PCR_ext;
            ...
            // 更新本地PCR时钟统计
            for (size_t i = 0; i < mPrograms.size(); ++i) {
                updatePCR(PID, PCR, byteOffsetFromStart);
            }
            ...
        }
        ...
    }
}
```

`ATSParser::parsePID()`方法开始根据PID对PAT与PMT进行解析，详细解析如下注释：
```c++
status_t ATSParser::parsePID(
        ABitReader *br, unsigned PID,
        unsigned continuity_counter,
        unsigned payload_unit_start_indicator) {
    // 首先根据PID找出PSI，目前只用到了PAT与PMT这两类PSI
    // 其中PAT的PID固定为0，在ATSParser初始化初期已经add进去
    ssize_t sectionIndex = mPSISections.indexOfKey(PID);

    if (sectionIndex >= 0) {
        const sp<PSISection> &section = mPSISections.valueAt(sectionIndex);
        ...
        ABitReader sectionBits(section->data(), section->size());
        // 若PID为0，则以PAT的格式进行解析
        if (PID == 0) {
            parseProgramAssociationTable(&sectionBits);
        } else {
            // 否则将以PMT的格式进行解析
            bool handled = false;
            for (size_t i = 0; i < mPrograms.size(); ++i) {
                status_t err;
                // parsePSISection()会调用parseProgramMap()解析PMT
                if (!mPrograms.editItemAt(i)->parsePSISection(
                            PID, &sectionBits, &err)) {
                    continue;
                }
                ...
            }
            // 处理成功则移除PID对应的PSISections
            if (!handled) {
                mPSISections.removeItem(PID);
            }
        }
        ...
    }
    // 遍历所有节目，对音视频的ES流进行解析
    bool handled = false;
    for (size_t i = 0; i < mPrograms.size(); ++i) {
        status_t err;
        if (mPrograms.editItemAt(i)->parsePID(
                    PID, continuity_counter, payload_unit_start_indicator,
                    br, &err)) {
            if (err != OK) {
                return err;
            }

            handled = true;
            break;
        }
    }
    ...
}
```

我们先来分析下PAT的解析过程，主要是完成`[节目编号->PID]`的映射解析，详见注释：
```c++
void ATSParser::parseProgramAssociationTable(ABitReader *br) {
    ... // 此处省略PAT常规字段的解析
    // 遍历节目列表进行[节目编号->PID]的映射解析
    for (size_t i = 0; i < numProgramBytes / 4; ++i) {
        unsigned program_number = br->getBits(16);
        ALOGV("    program_number = %u", program_number);

        MY_LOGV("    reserved = %u", br->getBits(3));
        // 节目编号0的时候为network_PID，这里没有用到，忽略
        if (program_number == 0) {
            MY_LOGV("    network_PID = 0x%04x", br->getBits(13));
        } else {
            // 开始对应[节目编号->PID]的映射解析
            unsigned programMapPID = br->getBits(13);

            ALOGV("    program_map_PID = 0x%04x", programMapPID);

            bool found = false;
            for (size_t index = 0; index < mPrograms.size(); ++index) {
                const sp<Program> &program = mPrograms.itemAt(index);

                if (program->number() == program_number) {
                    program->updateProgramMapPID(programMapPID);
                    found = true;
                    break;
                }
            }
            // 添加节目列表
            if (!found) {
                mPrograms.push(
                        new Program(this, program_number, programMapPID));
            }
            // 添加节目对应的PSISection
            if (mPSISections.indexOfKey(programMapPID) < 0) {
                mPSISections.add(programMapPID, new PSISection);
            }
        }
    }
    ...
}
```

紧接着是PMT的解析，主要是完成`[ES流->PID]`的映射解析，并创建对应的`Stream`对象，详见注释：
```c++
status_t ATSParser::Program::parseProgramMap(ABitReader *br) {
    ... // 此处省略PMT常规字段的解析
    // infoBytesRemaining is the number of bytes that make up the
    // variable length section of ES_infos. It does not include the
    // final CRC.
    size_t infoBytesRemaining = section_length - 9 - program_info_length - 4;
    // 遍历所有Stream，完成[ES流->PID]的映射解析
    while (infoBytesRemaining > 0) {
        CHECK_GE(infoBytesRemaining, 5u);
        // 这里最重要的就是streamType与PID的解析
        unsigned streamType = br->getBits(8);
        ALOGV("    stream_type = 0x%02x", streamType);
        ...
        unsigned elementaryPID = br->getBits(13);
        ALOGV("    elementary_PID = 0x%04x", elementaryPID);
        ...
        // 添加StreamInfo到列表中
        StreamInfo info;
        info.mType = streamType;
        info.mPID = elementaryPID;
        infos.push(info);

        infoBytesRemaining -= 5 + ES_info_length;
    }
    ... // 此处省略PID改变导致的异常的处理，基本可以忽略

    // 遍历StreamInfo，根据PID添加对应的音视频Stream
    for (size_t i = 0; i < infos.size(); ++i) {
        StreamInfo &info = infos.editItemAt(i);

        ssize_t index = mStreams.indexOfKey(info.mPID);

        if (index < 0) {
            sp<Stream> stream = new Stream(
                    this, info.mPID, info.mType, PCR_PID);

            mStreams.add(info.mPID, stream);
        }
    }

    return OK;
}
```

然后就到了音视频流的解析，调用对应`Stream`的`parse()`方法开始进行解析：
```c++
bool ATSParser::Program::parsePID(
        unsigned pid, unsigned continuity_counter,
        unsigned payload_unit_start_indicator,
        ABitReader *br, status_t *err) {
    *err = OK;

    ssize_t index = mStreams.indexOfKey(pid);
    if (index < 0) {
        return false;
    }
    // 根据PID查找出对应的Stream，开始解析
    *err = mStreams.editValueAt(index)->parse(
            continuity_counter, payload_unit_start_indicator, br);

    return true;
}
```

`Stream`解析的过程中，最重要的是`payload_unit_start_indicator`这个参数。前面我们说到，要获得一帧完整的数据，就需要把连续几个TS包里的Payload数据全部取出来，才能组合成一个PES包。那么`payload_unit_start_indicator`值为1时代表一个完整的音视频数据包的开始。那么从这里开始，直到下一个值为1的包为止（相同PID的ES流），把所有的这些TS包组合起来就是一个完整的PES包。了解了这个要点，看下面的代码就比较简单了：
```c++
status_t ATSParser::Stream::parse(
        unsigned continuity_counter,
        unsigned payload_unit_start_indicator, ABitReader *br) {
    ...
    if (payload_unit_start_indicator) {
        // payload_unit_start_indicator值为1并且之前已经started了，则触发flush()开始解析PES
        // 因为这个时候已经组装完一个完成的PES包了
        if (mPayloadStarted) {
            // Otherwise we run the danger of receiving the trailing bytes
            // of a PES packet that we never saw the start of and assuming
            // we have a a complete PES packet.

            status_t err = flush();

            if (err != OK) {
                return err;
            }
        }

        mPayloadStarted = true;
    }

    if (!mPayloadStarted) {
        return OK;
    }
    ... // 这部分省略的代码主要是对PES的buffer进行组装

    return OK;
}
```

`flush()`操作调用了`parsePES()`开始对PES包进行解析，然后解析出`PES data`，详细解析如下注释：
```c++
status_t ATSParser::Stream::parsePES(ABitReader *br) {
    ...
    // 解析Stream ID
    unsigned stream_id = br->getBits(8);
    ALOGV("stream_id = 0x%02x", stream_id);
    // 获取PES包的长度，注意这里的长度为后续Payload长度，不包含自身，单位为字节
    unsigned PES_packet_length = br->getBits(16);
    ALOGV("PES_packet_length = %u", PES_packet_length);
    // 过滤掉非音视频PES，开始解析
    if (stream_id != 0xbc  // program_stream_map
            && stream_id != 0xbe  // padding_stream
            && stream_id != 0xbf  // private_stream_2
            && stream_id != 0xf0  // ECM
            && stream_id != 0xf1  // EMM
            && stream_id != 0xff  // program_stream_directory
            && stream_id != 0xf2  // DSMCC
            && stream_id != 0xf8) {  // H.222.1 type E
        ... // 省略部分Header字段解析
        if (PTS_DTS_flags == 2 || PTS_DTS_flags == 3) {
            ... // 解析PTS，其中2代表'10'，只有PTS；3代表'11'，PTS与DTS都有
            ALOGV("PTS = 0x%016llx (%.2f)", PTS, PTS / 90000.0);

            optional_bytes_remaining -= 5;

            if (PTS_DTS_flags == 3) {
                ... // 解析DTS
                ALOGV("DTS = %llu", DTS);

                optional_bytes_remaining -= 5;
            }
        }
        ... // 省略部分Header字段解析
        // ES data follows.
        // 开始解析PES data
        if (PES_packet_length != 0) {
            CHECK_GE(PES_packet_length, PES_header_data_length + 3);
            // PES data大小：包长度 - PES固定头长度3 - Header的data长度
            unsigned dataLength =
                PES_packet_length - 3 - PES_header_data_length;
            // 对剩余字节数进行校验，确保与上面的dataLength一致
            if (br->numBitsLeft() < dataLength * 8) {
                ALOGE("PES packet does not carry enough data to contain "
                     "payload. (numBitsLeft = %d, required = %d)",
                     br->numBitsLeft(), dataLength * 8);

                return ERROR_MALFORMED;
            }

            CHECK_GE(br->numBitsLeft(), dataLength * 8);
            // 通过onPayloadData()回调音视频PES data
            onPayloadData(
                    PTS_DTS_flags, PTS, DTS, br->data(), dataLength);

            br->skipBits(dataLength * 8);
        } else {
            // PES_packet_length为0的情况下，把剩余的字节当做PES data
            onPayloadData(
                    PTS_DTS_flags, PTS, DTS,
                    br->data(), br->numBitsLeft() / 8);

            size_t payloadSizeBits = br->numBitsLeft();
            CHECK_EQ(payloadSizeBits % 8, 0u);

            ALOGV("There's %d bytes of payload.", payloadSizeBits / 8);
        }
    }
    ...
    return OK;
}
```

## 总结
最终，通过`onPayloadData()`回调音视频裸流给`MediaPlayer`进行解码，进行音视频数据的播放，整个`Native Sink`端的流程就到此结束了。相信看完上面所有源码解析后，自己写这部分逻辑也不是难事，当然更好的办法肯定是基于Sink端的代码进行移植。

移植`Native Sink`的难点主要是对Native相关的依赖代码进行隔离，如：`ALooper`与`AHandler`异步消息机制、`ANetworkSession`网络连接部分与`foundation`包下的相关的实现等，且移植需要一定的C/C++与NDK开发能力。这里建议可以通过Android应用层实现RTSP连接（Socket/Netty）、音视频解码（MediaCodec/FFmpeg）与渲染（SurfaceView/TextureView），然后把底层的RTP、MPEG2-TS解析部分的代码进行移植，这样可以大大减少Native相关的依赖，提高移植效率。