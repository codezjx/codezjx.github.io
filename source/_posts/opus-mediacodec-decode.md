---
title: Android上使用MediaCodec对Opus音频解码
date: 2019-09-04 20:10:34
categories: Opus
tags: [Android, Opus, MediaCodec]
---

## 简介
`Opus`是一个有损声音编码的格式，由Xiph.Org基金会开发，之后由互联网工程任务组进行标准化，目标是希望用单一格式包含声音和语音，取代Speex和Vorbis，且适用于网络上低延迟的即时声音传输，标准格式定义于RFC 6716文件。`Opus`格式是一个开放格式，使用上没有任何专利或限制。

在官方的[examples][1]测试案例中，就算有30%的丢包率，也基本能够听清楚人声，这对于即时的声音传输场景来说非常重要。并且它支持动态、无缝的调节比特率与音频带宽，在网络环境多变的场景下更能保证音频的质量。

同时它是`WebRTC`中默认的音频编码格式、也是`WebM`视频文件中音频的编码格式。由于最近项目中需要对`Opus`音频流进行解码，遇到了很多问题，网上基本也搜不到相关的资料与demo，因此诞生了这篇文章，希望能够帮助到有需要的同学。

## 技术特性
`Opus`可以处理各种音频应用，包括IP语音、视频会议、游戏内聊天、流音乐、甚至远程现场音乐表演。它可以从低比特率窄带语音扩展到非常高清音质的立体声音乐。支持的功能包括：

- 6 kb/秒到510 kb/秒的比特率；单一频道最高256 kb/秒
- 采样率从8 kHz（窄带）到48 kHz（全频）
- 帧大小从2.5毫秒到60毫秒
- 支持恒定比特率（CBR）、受约束比特率（CVBR）和可变比特率（VBR）
- 支持语音（SILK层）和音乐（CELT层）的单独或混合模式
- 支持单声道和立体声；支持多达255个音轨（多数据流的帧）
- 可动态调节比特率，音频带宽和帧大小
- 良好的鲁棒性丢失率和数据包丢失隐藏（PLC）
- 浮点和定点实现

## MediaCodec解码
从官方的[media-formats][2]文档中我们可以知道，从`Android 5.0+`开始，官方就已经添加了`Opus`原生的编解码支持，我们使用`MediaCodec`就能够对其进行编解码的实现。关于`MediaCodec`的使用就不在这里做介绍了，大家可以参考下[官方介绍文档][3]，非常的详细。

于是乎很快就撸起Android Studio开始编写解码相关的代码，最开始初始化代码是这样子的，通过一些基本参数如：`sampleRate`和`channelCount`初始化`MediaCodec`并启动：
```java
final int sampleRate = 48000;
final int channelCount = 2;
final int bitRate = 102000;
MediaFormat mediaFormat = MediaFormat.createAudioFormat(MediaFormat.MIMETYPE_AUDIO_OPUS, sampleRate, channelCount);
mediaFormat.setInteger(MediaFormat.KEY_BIT_RATE, bitRate);
MediaCodec mediaCodec = MediaCodec.createDecoderByType(MediaFormat.MIMETYPE_AUDIO_OPUS);
mediaCodec.configure(mediaFormat, null, null, 0);
mediaCodec.start();
```

之后就是通过`queueInputBuffer()`将音频流数据送进`MediaCodec`进行解码，然后通过`dequeueOutputBuffer`从缓冲区拿到解码后的数据，这里代码先忽略，还不是重点。当第一帧数据通过`queueInputBuffer()`进入`MediaCodec`的时候，就会报以下的错误：
```
2019-09-04 19:48:36.701 17584-17678/com.codezjx.opusdecode E/ACodec: [OMX.google.opus.decoder] ERROR(0x80001001)
2019-09-04 19:48:36.701 17584-17678/com.codezjx.opusdecode E/ACodec: signalError(omxError 0x80001001, internalError -2147483648)
2019-09-04 19:48:36.701 17584-17678/com.codezjx.opusdecode E/MediaCodec: Codec reported err 0x80001001, actionCode 0, while in state 6
```

紧接着调用`dequeueOutputBuffer()`取出解码后数据的时候，就会报`IllegalStateException`，表明现在`MediaCodec`不在执行状态，也就是说上面解码的过程出错，导致`MediaCodec`的状态异常了。
```
    --------- beginning of crash
2019-09-04 19:48:36.702 17584-17677/com.codezjx.opusdecode E/AndroidRuntime: FATAL EXCEPTION: Thread-7
    Process: com.codezjx.opusdecode, PID: 17584
    java.lang.IllegalStateException
        at android.media.MediaCodec.native_dequeueOutputBuffer(Native Method)
        at android.media.MediaCodec.dequeueOutputBuffer(MediaCodec.java:2698)
```

这个时候就比较纳闷了，设备已经是`Android 5.0+`，应该是支持`Opus`解码的。猜测应该是参数设置有问题，因此又回去看[MediaCodec][3]的官方文档，果然发现漏了一些很重要的参数：[CSD buffer][4]。

## Codec-specific Data
对于某些格式，特别是AAC音频和MPEG4、H.264和H.265视频格式，要求实际数据需要以多个缓冲区为前缀，包含设置数据或编解码器特定数据。下图是官方对于不同格式需要设置的不同`CSD buffer`数据的对照图，其中`Opus`音频解码需要在`CSD buffer #0/#1/#2`三个缓冲区分别设置不同的参数。

![](csd-buffer.jpg)

设置参数的方式有两种，第一种比较复杂，需要在`MediaCodec.start()`之后，且在解码任何数据帧之前，将这些数据提交给`MediaCodec`，需要通过`BUFFER_FLAG_CODEC_CONFIG`这个flag进行标记：
```java
mediaCodec.queueInputBuffer(inputIdex, 0, bytes.length, 0, MediaCodec.BUFFER_FLAG_CODEC_CONFIG);
```

第二种方式就简单很多，直接通过`MediaFormat.setByteBuffer()`方法，就可以非常简单的对这些缓冲区进行赋值，推荐第二种方式：
```java
ByteBuffer csd0 = ByteBuffer.wrap(bytes);
mediaFormat.setByteBuffer("csd-0", csd0);
```

其中`Pre-skip`和`Pre-roll`参数目前还没有用到，因此设置默认值0应该就可以了，注意这里格式需要是64位的无符号整数，并且是nativeOrder，Android上一般就是小端模式。但是官方文档中对于`csd-0`需要设置的参数就没有详细介绍了，只写明了是`Identification header`，也就是标识头，因此我们需要对这个标识头再进行详细分析。

## Opus Identification header
关于Opus的标识头，可以从[rfc7845][5]标准文档中找到详细的解答，格式如下：
```
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |      'O'      |      'p'      |      'u'      |      's'      |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |      'H'      |      'e'      |      'a'      |      'd'      |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |  Version = 1  | Channel Count |           Pre-skip            |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                     Input Sample Rate (Hz)                    |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |   Output Gain (Q7.8 in dB)    | Mapping Family|               |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+               :
 |                                                               |
 :               Optional Channel Mapping Table...               :
 |                                                               |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

                    Figure 2: ID Header Packet
```
`Identification header`主要由8个参数组成，分别为：
- Magic Signature：固定头，占8个字节，为字符串OpusHead
- Version：版本号，占1字节，固定为0x01
- Channel Count：通道数，占1字节，根据音频流通道自行设置，如0x02
- Pre-skip：回放的时候从解码器中丢弃的samples数量，占2字节，为小端模式，默认设置0x00, 0x00就好
- Input Sample Rate (Hz)：音频流的Sample Rate，占4字节，为小端模式，根据实际情况自行设置
- Output Gain：输出增益，占2字节，为小端模式，没有用到默认设置0x00, 0x00就好
- Channel Mapping Family：通道映射系列，占1字节，默认设置0x00就好
- Channel Mapping Table：可选参数，上面的Family默认设置0x00的时候可忽略

最终`MediaCodec`的初始化代码如下，增加了对`csd-0/1/2`3个缓冲区的设置：

```java
final int sampleRate = 48000;
final int channelCount = 2;
final int bitRate = 102000;
MediaFormat mediaFormat = MediaFormat.createAudioFormat(MediaFormat.MIMETYPE_AUDIO_OPUS, sampleRate, channelCount);
mediaFormat.setInteger(MediaFormat.KEY_BIT_RATE, bitRate);
byte[] csd0bytes = {
        // Opus
        0x4f, 0x70, 0x75, 0x73,
        // Head
        0x48, 0x65, 0x61, 0x64,
        // Version
        0x01,
        // Channel Count
        0x02,
        // Pre skip
        0x00, 0x00,
        // Input Sample Rate (Hz), eg: 48000
        (byte) 0x80, (byte) 0xbb, 0x00, 0x00,
        // Output Gain (Q7.8 in dB)
        0x00, 0x00,
        // Mapping Family
        0x00};
byte[] csd1bytes = {0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00};
byte[] csd2bytes = {0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00};
ByteBuffer csd0 = ByteBuffer.wrap(csd0bytes);
mediaFormat.setByteBuffer("csd-0", csd0);
ByteBuffer csd1 = ByteBuffer.wrap(csd1bytes);
mediaFormat.setByteBuffer("csd-1", csd1);
ByteBuffer csd2 = ByteBuffer.wrap(csd2bytes);
mediaFormat.setByteBuffer("csd-2", csd2);
MediaCodec mediaCodec = MediaCodec.createDecoderByType(MediaFormat.MIMETYPE_AUDIO_OPUS);
mediaCodec.configure(mediaFormat, null, null, 0);
mediaCodec.start();
```

紧接着通过`queueInputBuffer()`将音频流数据送进`MediaCodec`进行解码，然后通过`dequeueOutputBuffer`从缓冲区拿到解码后的数据，最后将数据写到`AudioTrack`中进行播放，即可完成`Opus`解码与声音回放。
```java
MediaCodec.BufferInfo decodeBufferInfo = new MediaCodec.BufferInfo();
while (isRunning) {
    int inputBufferId = mediaCodec.dequeueInputBuffer(timeoutUs);
    if (inputBufferId >= 0) {
        ByteBuffer inputBuffer = mediaCodec.getInputBuffer(inputBufferId);
        inputBuffer.clear();
        byte[] data = mDataQueue.take();
        inputBuffer.put(data, 0, data.length);
        mediaCodec.queueInputBuffer(inputBufferId, 0, data.length, 0, 0);
    }
    int outputBufferId = mediaCodec.dequeueOutputBuffer(decodeBufferInfo, timeoutUs);
    if (outputBufferId >= 0) {
        ByteBuffer outputBuffer = mediaCodec.getOutputBuffer(outputBufferId);
        byte[] pcm = new byte[decodeBufferInfo.size];
        outputBuffer.get(pcm);
        outputBuffer.clear();
        mAudioTrack.write(pcm, 0, pcm.length);
        mediaCodec.releaseOutputBuffer(outputBufferId, false);
    }
}
```

[1]: https://opus-codec.org/examples/
[2]: https://developer.android.com/guide/topics/media/media-formats
[3]: https://developer.android.com/reference/android/media/MediaCodec
[4]: https://developer.android.com/reference/android/media/MediaCodec#CSD
[5]: https://tools.ietf.org/html/rfc7845#section-5.1