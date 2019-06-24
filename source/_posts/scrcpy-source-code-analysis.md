---
title: scrcpy投屏工具源码解析
date: 2019-06-24 21:11:36
categories: Source Code Analysis
tags: [Android, scrcpy, RTFSC]
---

说到Android上投屏到PC或Mac端的软件，相信大家对Vysor应该都比较熟悉了，它是Chrome上的一款插件，安装完后就能方便的进行手机的投屏，但这款软件现在Pro版要收费了，免费版还一堆的广告。

今天我们要介绍的的是Genymobile自家开源的[scrcpy][1]工具，它也能实现我们想要的投屏功能，而且完全免费，没有任何广告，更重要的是它是一款开源软件！开源软件！开源软件！重要的事情说3遍。

在Mac上，通过Homebrew进行安装，等所有依赖下载完就可以使用了。
```
brew install scrcpy
```

通过adb connect或者USB数据线连接上电脑后，就可以进行投屏了，直接命令行运行`scrcpy`即可，Mac上运行起来的效果如下图。相比与Vysor，scrcpy是不需要安装任何APK的，方便快捷！
![](scrcpy.jpg)

可是，如果手机上不安装个发送端，那么Mac接收端怎么接收录屏数据呢？这个时候，强烈的好奇心驱使我挖掘`scrcpy`的源码。让我们带着这个疑问，一步步的从源码中寻找答案。

## 从编译scrcpy说起
打开源码下的`BUILD.md`介绍文件，里面会有各个平台的的详细编译流程。

这里面分client和server端，其中client端是展示端，接收录屏数据并进行解码显示。源码位于`app`目录。全是c代码，里面用到了FFmpeg和SDL库，都是跨平台的解决方案。因此通过[meson][2]进行交叉编译后，就能实现展示端的跨平台，支持Linux/Windows/Mac OS/Docker等平台。

>Tips：Meson是用于自动化构建的自由软件，使用Python语言编写，在Apache许可证 2.0版本下发布，主要目标是为了让开发者节约用于配置构建系统的时间。

server端就是发送端了，也就是我们的Android手机端，进行录屏后编码发送给展示端进行展示。源码位于`server`目录，下文我们的重点也会放在Android端源码的分析上面。奇怪了，不是说手机端上是没有安装任何apk的吗？那么这部分代码是怎么运行在Android平台上的呢？

继续打开`server/meson.build`文件查看server的编译流程（跟`build.gradle`一样，`meson.build`是`meson`编译所必须的脚本文件）
```meson
# It may be useful to use a prebuilt server, so that no Android SDK is required
# to build. If the 'prebuilt_server' option is set, just copy the file as is.
prebuilt_server = get_option('prebuilt_server')
if prebuilt_server == ''
    custom_target('scrcpy-server',
                  build_always: true,  # gradle is responsible for tracking source changes
                  output: 'scrcpy-server.jar',
                  command: [find_program('./scripts/build-wrapper.sh'), meson.current_source_dir(), '@OUTPUT@', get_option('buildtype')],
                  install: true,
                  install_dir: 'share/scrcpy')
else
    if not prebuilt_server.startswith('/')
        # relative path needs some trick
        prebuilt_server = meson.source_root() + '/' + prebuilt_server
    endif
    custom_target('scrcpy-server-prebuilt',
                  input: prebuilt_server,
                  output: 'scrcpy-server.jar',
                  command: ['cp', '@INPUT@', '@OUTPUT@'],
                  install: true,
                  install_dir: 'share/scrcpy')
endif

```

可以看到里面通过运行`build-wrapper.sh`调用`gradle`编译了apk，并将apk最终输出名字改成了`scrcpy-server.jar`。
```shell
...
if [[ "$BUILDTYPE" == debug ]]
then
    "$GRADLE" -p "$PROJECT_ROOT" assembleDebug
    cp "$PROJECT_ROOT/build/outputs/apk/debug/server-debug.apk" "$OUTPUT"
else
    "$GRADLE" -p "$PROJECT_ROOT" assembleRelease
    cp "$PROJECT_ROOT/build/outputs/apk/release/server-release-unsigned.apk" "$OUTPUT"
fi
```

## 执行scrcpy
上面编译生成的`scrcpy-server.jar`应该就是server端了，那它又是怎么在Android平台上运行起来的呢？通过跟踪client代码发现（源码在`server.c`），在scrcpy运行的时候，会执行`push_server()`函数，将`scrcpy-server.jar`通过`adb push`到Android的`/data/local/tmp/scrcpy-server.jar`目录下。
```c
...
#define SERVER_FILENAME "scrcpy-server.jar"
#define DEVICE_SERVER_PATH "/data/local/tmp/" SERVER_FILENAME
...
static bool
push_server(const char *serial) {
    process_t process = adb_push(serial, get_server_path(), DEVICE_SERVER_PATH);
    return process_check_success(process, "adb push");
}
```

然后通过`server.c`中的`execute_server()`方法，执行`app_process`指令将server端运行起来。
```c
static process_t
execute_server(struct server *server, const struct server_params *params) {
    char max_size_string[6];
    char bit_rate_string[11];
    sprintf(max_size_string, "%"PRIu16, params->max_size);
    sprintf(bit_rate_string, "%"PRIu32, params->bit_rate);
    const char *const cmd[] = {
        "shell",
        "CLASSPATH=/data/local/tmp/" SERVER_FILENAME,
        "app_process",
        "/", // unused
        "com.genymobile.scrcpy.Server",
        max_size_string,
        bit_rate_string,
        server->tunnel_forward ? "true" : "false",
        params->crop ? params->crop : "-",
        params->send_frame_meta ? "true" : "false",
        params->control ? "true" : "false",
    };
    return adb_execute(server->serial, cmd, sizeof(cmd) / sizeof(cmd[0]));
}
```

## 关于app_process
上面我们提到`scrcpy-server.jar`是通过`app_process`运行起来的，那么`app_process`又是什么鬼？`app_process`是启动zygote和其他Java程序的应用程序，它可以让虚拟机从main()方法开始执行一个Java 程序。具体的用法可以参考[app_process][3]源码中的注释：
```cpp
"Usage: app_process [java-options] cmd-dir start-class-name [options]\n");
```

与APP进程不同，通过`app_process`启动的进程可以在root权限和shell权限 (adb默认)下启动，也就分别拥有了调用不同API的能力。通常情况下shell权限启动的`app_process` 只能够调用一些能够完成adb本身工作的API，root权限启动的`app_process`进程则拥有更多权限，甚至能够调用系统`signature`保护级别的API及访问整个文件系统。

实际上不少adb命令都是对调用`app_process`进行了一些封装，这里举我们平时常用的[am][4]指令为例，我们在执行`adb shell am`的时候其实是执行了以下的脚本。通过`app_process`运行`am.jar`中`com.android.commands.am.Am`这个类的`main()`函数来完成具体操作。
```
#!/system/bin/sh
if [ "$1" != "instrument" ] ; then
    cmd activity "$@"
else
    base=/system
    export CLASSPATH=$base/framework/am.jar
    exec app_process $base/bin com.android.commands.am.Am "$@"
fi
```

这里需要注意的是`app_process`只能运行原始dex文件，也可以接收包含`classes.dex`的jar包，如`am.jar`。这里`scrcpy`取巧直接用了编译apk的方式，再直接重命名为jar包，这样也可以被`app_process`运行。而且可以使用gradle方便的进行编译，直接在Android Studio中开发，调用原生API，像源码中的`IRotationWatcher.aidl`也可以直接生成相应的IPC Proxy类进行调用，省力又省心。

`scrcpy-server.jar`在adb下通过`app_process`运行起来后，默认就有了shell权限，像一些截屏、录屏、模拟按键点击等功能都可以直接使用，而且完全不需要任何权限的声明。

## RTFSC
了解了编译与启动的方式后，我们终于可以开始分析`scrcpy`的源码了，从发送端的主入口`com.genymobile.scrcpy.Server.main()`方法开始看起。
```java
public static void main(String... args) throws Exception {
    ...
    unlinkSelf();
    // 对尺寸、码率、裁剪等参数进行解析，初始化Options对象
    Options options = createOptions(args);
    scrcpy(options);
}

private static void scrcpy(Options options) throws IOException {
    final Device device = new Device(options);
    boolean tunnelForward = options.isTunnelForward();
    // 根据tunnelForward的值来创建连接
    try (DesktopConnection connection = DesktopConnection.open(device, tunnelForward)) {
        ScreenEncoder screenEncoder = new ScreenEncoder(options.getSendFrameMeta(), options.getBitRate());
        // 根据Control参数确认是否能对设备进行操作，如按键、鼠标等事件的响应
        if (options.getControl()) {
            Controller controller = new Controller(device, connection);
            // asynchronous
            // 开了2条线程，通过异步的方式分别启动controller和sender
            startController(controller);
            startDeviceMessageSender(controller.getSender());
        }

        try {
            // synchronous
            // 同步的方式录屏、编码并发送数据到展示端
            screenEncoder.streamScreen(device, connection.getVideoFd());
        } catch (IOException e) {
            // this is expected on close
            Ln.d("Screen streaming stopped");
        }
    }
}
```

其中通过`DesktopConnection.open()`方法根据`tunnelForward`进行发送端与接收端的服务连接。如果是`adb forward`设置端口转发的方式，则发送端作为服务器，初始化`LocalServerSocket`进行监听；否则若是`adb reverse`反向代理的方式，则通过`LocalSocket`链接到远程服务器，此时接收端作为服务器。关于`adb forward`和`adb reverse`的使用方式不在这里展开，有兴趣的可以自己搜索相关用法。
```java
public static DesktopConnection open(Device device, boolean tunnelForward) throws IOException {
    LocalSocket videoSocket;
    LocalSocket controlSocket;
    if (tunnelForward) {
        // adb forward模式下作为服务器进行监听
        LocalServerSocket localServerSocket = new LocalServerSocket(SOCKET_NAME);
        try {
            videoSocket = localServerSocket.accept();
            ...
            try {
                controlSocket = localServerSocket.accept();
            } catch (IOException | RuntimeException e) {
                videoSocket.close();
                throw e;
            }
        } finally {
            localServerSocket.close();
        }
        // adb reverse模式下作为客户端链接到服务器
    } else {
        videoSocket = connect(SOCKET_NAME);
        try {
            controlSocket = connect(SOCKET_NAME);
        } catch (IOException | RuntimeException e) {
            videoSocket.close();
            throw e;
        }
    }
    ...
    return connection;
}
```

关于`tunnelForward`的值，在展示端源码`server.c`的`enable_tunnel`中，会尝试执行`adb reverse`指令，如果失败了则改为`adb forward`进行端口转发的方式，从实现上来看主要是发送端作为客户端还是服务端的区别。
```c
static bool
enable_tunnel(struct server *server) {
    if (enable_tunnel_reverse(server->serial, server->local_port)) {
        return true;
    }

    LOGW("'adb reverse' failed, fallback to 'adb forward'");
    server->tunnel_forward = true;
    return enable_tunnel_forward(server->serial, server->local_port);
}
```

当连接建立起来后，就可以通过`ScreenEncoder.streamScreen()`方法开始进行录屏、编码与数据的发送了，以下为精简后的核心逻辑代码与流程解析。
```java
public void streamScreen(Device device, FileDescriptor fd) throws IOException {
    ...
    do {
        // 首先通过MediaCodec创建了一个H.264类型的编码器
        MediaCodec codec = MediaCodec.createEncoderByType("video/avc");
        // 通过反射SurfaceControl创建了一个虚拟显示
        IBinder display = SurfaceControl.createDisplay("scrcpy", true);
        ...
        codec.configure(format, null, null, MediaCodec.CONFIGURE_FLAG_ENCODE);
        // 通过createInputSurface()获取编码器的输入Surface
        // 后续将这块Surface的内容作为编码器的输入数据源
        Surface surface = codec.createInputSurface();
        // 将Surface和虚拟显示进行绑定，录屏数据将输出到这块Surface上
        setDisplaySurface(display, surface, contentRect, videoRect);
        codec.start();
        try {
            // 进行编码并发送数据
            alive = encode(codec, fd);
            // do not call stop() on exception, it would trigger an IllegalStateException
            codec.stop();
        } finally {
            ...
        }
    } while (alive);
    ...
}

// 通过反射SurfaceControl并调用一系列方法，初始化录屏相关的环境，并与Surface进行绑定
private static void setDisplaySurface(IBinder display, Surface surface, Rect deviceRect, Rect displayRect) {
    SurfaceControl.openTransaction();
    try {
        SurfaceControl.setDisplaySurface(display, surface);
        SurfaceControl.setDisplayProjection(display, 0, deviceRect, displayRect);
        SurfaceControl.setDisplayLayerStack(display, 0);
    } finally {
        SurfaceControl.closeTransaction();
    }
}
```
>注意：这里的`SurfaceControl`是对系统`android.view.SurfaceControl`的反射包装类。因为这个类属于系统隐藏API，客户端无法直接调用，`wrappers`包下的类基本全都是通过这种方式进行调用的。

了解完录屏数据的获取以及如何与编码器绑定后，我们来看下怎么拿到编码后的数据及发送，这个时候来看下`encode()`方法。
```java
private boolean encode(MediaCodec codec, FileDescriptor fd) throws IOException {
    ...
    while (!consumeRotationChange() && !eof) {
        // 通过dequeueOutputBuffer()从输出缓存队列中取出buffer
        int outputBufferId = codec.dequeueOutputBuffer(bufferInfo, -1);
        // 若flag为BUFFER_FLAG_END_OF_STREAM代表到了流的结尾处，这个时候该停止编码
        // eof将置为true并停止while循环
        eof = (bufferInfo.flags & MediaCodec.BUFFER_FLAG_END_OF_STREAM) != 0;
        try {
            ...
            if (outputBufferId >= 0) {
                // 根据出队的buffer id去获取指定的ByteBuffer对象
                ByteBuffer codecBuffer = codec.getOutputBuffer(outputBufferId);
                if (sendFrameMeta) {
                    writeFrameMeta(fd, bufferInfo, codecBuffer.remaining());
                }
                // 此时buffer中的数据已经是编码后的H.264包数据，发送到展示端即可
                IO.writeFully(fd, codecBuffer);
            }
        } finally {
            if (outputBufferId >= 0) {
                // 处理完编码后的数据后需要及时释放buffer
                codec.releaseOutputBuffer(outputBufferId, false);
            }
        }
    }
    return !eof;
}
```

看到这里，发送端的录屏数据采集、编码及发送就结束了。剩下的流程就是展示端接收并通过FFmpeg解码H.264数据，再通过SDL展示每一帧数据的过程了。未完，待续...


[1]: https://github.com/Genymobile/scrcpy
[2]: https://mesonbuild.com/
[3]: https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/cmds/app_process/app_main.cpp
[4]: https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/cmds/am/am
