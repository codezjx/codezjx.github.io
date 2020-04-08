---
title: Miracast技术详解（一）：Wi-Fi Display
date: 2020-02-04 10:20:33
categories: Miracast
tags: [Android, Miracast, Wi-Fi Display]
---

## Miracast概述

### Miracast
Miracast是由Wi-Fi联盟于2012年所制定，以Wi-Fi直连（Wi-Fi Direct）为基础的无线显示标准。支持此标准的消费性电子产品（又称3C设备）可透过无线方式分享视频画面，例如手机可透过Miracast将影片或照片直接在电视或其他设备播放而无需任何连接线，也不需透过无线热点（AP，Access Point）。

### Wi-Fi Direct
Wi-Fi直连（英语：Wi-Fi Direct），之前曾被称为Wi-Fi点对点（Wi-Fi Peer-to-Peer），是一套无线网络互连协议，让wifi设备可以不必透过无线网络接入点（Access Point），以点对点的方式，直接与另一个wifi设备连线，进行高速数据传输。这个协议由Wi-Fi联盟发展、支持与授与认证，通过认证的产品将可获得Wi-Fi CERTIFIED Wi-Fi Direct®标志。

### Wi-Fi Display
Wi-Fi Display是Wi-Fi联盟制定的一个标准协议，它结合了Wi-Fi标准和H.264视频编码技术。利用这种技术，消费者可以从一个移动设备将音视频内容实时镜像到大型屏幕，随时、随地、在各种设备之间可靠地传输和观看内容。

Miracast实际上就是Wi-Fi联盟对支持WiFi Display功能的设备的认证名称，产品通过认证后会打上Miracast标签。

### Sink & Source
如下图所示，Miracast可分为发送端与接收端。Source端为Miracast音视频数据发送端，负责音视频数据的采集、编码及发送。而Sink端为Miracast业务的接收端，负责接收Source端的音视频码流并解码显示，其中通过Wi-Fi Direct技术进行连接。

![](wfd.jpg)

## Android上Wi-Fi Direct的实现

上面的概述里面也说到，Miracast是基于Wi-Fi Direct技术来实现连接与数据传输。那么要实现Miracast技术，首先就得研究下Android平台下的Wi-Fi Direct技术。

### Wi-Fi P2P 服务发现
Wi-Fi Direct（在Android平台上也称Wi-Fi P2P）,可以让具备相应硬件的Android 4.0（API 级别 14）或更高版本设备在没有AP的情况下，通过WLAN进行直接互联，主要的调用集中在`WifiP2pManager`这个类中。

#### 设置应用权限

首先在`Manifest`中加入以下权限，因为在P2P传输数据的时候会使用到Socket进行C/S连接，因此需要使用到INTERNET权限。
```xml
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.CHANGE_WIFI_STATE" />
<uses-permission android:name="android.permission.INTERNET" />
```

#### 初始化WifiP2pManager

获取系统P2P服务`WifiP2pManager`的实例，并通过调用`initialize()`获取`Channel`实例，后面绝大多数api的调用都将使用到`mManager`和`mChannel`。
```java
mManager = (WifiP2pManager) context.getSystemService(Context.WIFI_P2P_SERVICE);
if (mManager != null) {
    mChannel = mManager.initialize(context, context.getMainLooper(), mChannelListener);
}
```

这里要注意，若系统WiFi没有开启的情况下，需要先调用`WiFiManager`的`setWifiEnabled()`方法手动开启WiFi：
```java
mWifiManager = (WifiManager) context.getSystemService(Context.WIFI_SERVICE);
if (!mWifiManager.isWifiEnabled()) {
    mWifiManager.setWifiEnabled(true);
}
```

#### 接收关键广播

创建广播接收器接收以下关键广播，在P2P连接期间各种设备与连接状态的变化都会通过以下广播进行通知。对于每个Action的意义及处理详见以下注释：
```java
public class WiFiDirectReceiver extends BroadcastReceiver implements WifiP2pManager.PeerListListener {

    private static final String TAG = "WiFiDirectReceiver";
    private WifiP2pManager mManager;
    private Channel mChannel;

    public WiFiDirectReceiver(WifiP2pManager manager, Channel channel) {
        mManager = manager;
        mChannel = channel;
    }

    @Override
    public void onReceive(Context context, Intent intent) {
        String action = intent.getAction();
        if (WifiP2pManager.WIFI_P2P_STATE_CHANGED_ACTION.equals(action)) {
            // 当Wifi P2P在设备上启用或停用时广播，通过EXTRA_WIFI_STATE字段可以获取状态
            int state = intent.getIntExtra(WifiP2pManager.EXTRA_WIFI_STATE, WifiP2pManager.WIFI_P2P_STATE_DISABLED);
            boolean isEnabled = (WifiP2pManager.WIFI_P2P_STATE_ENABLED == state);
            ...
        } else if (WifiP2pManager.WIFI_P2P_PEERS_CHANGED_ACTION.equals(action)) {
            // 调用discoverPeers()后，若检测到对等设备时会广播
            // 指示P2P设备列表已有变化，可调用requestPeers()获取当前最新的对等设备列表
            if (mManager != null) {
                mManager.requestPeers(channel, this);
            }
            ...
        } else if (WifiP2pManager.WIFI_P2P_CONNECTION_CHANGED_ACTION.equals(action)) {
            // 当设备的Wifi P2P连接状态更改时广播
            // 通过EXTRA_NETWORK_INFO字段获取NetworkInfo，可判断P2P的连接状态
            // 并可通过requestConnectionInfo()、requestNetworkInfo()或requestGroupInfo()来检索当前连接信息
            NetworkInfo networkInfo = intent.getParcelableExtra(WifiP2pManager.EXTRA_NETWORK_INFO);
            if (networkInfo == null) {
                Log.e(TAG, "WifiP2pManager NetworkInfo info is null!");
                return;
            }
            if (networkInfo.isConnected()) {
                Log.d(TAG, "Wi-Fi Direct connected");
                // We are connected with the other device, request connection
                // info to find group owner IP
                manager.requestConnectionInfo(channel, connectionListener);
            } else {
            	Log.d(TAG, "Wi-Fi Direct disconnected");
            }
            ...
        } else if (WifiP2pManager.WIFI_P2P_THIS_DEVICE_CHANGED_ACTION.equals(action)) {
            // 当设备的详细信息（例如设备名称）更改时广播
            // 通过EXTRA_WIFI_P2P_DEVICE字段可获取当前设备信息WifiP2pDevice
            // 并可使用requestDeviceInfo()来检索当前设备信息
            WifiP2pDevice device = intent.getParcelableExtra(WifiP2pManager.EXTRA_WIFI_P2P_DEVICE);
            ...
        }
    }

    @Override
    public void onPeersAvailable(WifiP2pDeviceList peerList) {
        if (peerList != null) {
            Collection<WifiP2pDevice> deviceList = peerList.getDeviceList();
            if (!deviceList.isEmpty()) {
                for (WifiP2pDevice wifiP2pDevice : deviceList) {
                    Log.d(TAG, "peer device: " + wifiP2pDevice.toString());
                }
            }
        }
    }
}
```

#### 发现对等设备

当初始化完`WifiP2pManager`后，我们就可以开始设备的发现了。通过`discoverPeers()`和`stopPeerDiscovery()`方法，可以完成设备发现的启动与停止。注意这两个方法会立即返回，并在对应的`ActionListener`中回调成功与否的通知。
```java
mManager.discoverPeers(mChannel, new WifiP2pManager.ActionListener() {
    @Override
    public void onSuccess() {
        Log.d(TAG, "onSuccess: searchDevice");
    }

    @Override
    public void onFailure(int reason) {
        Log.d(TAG, "onFailure: searchDevice" + reason);
    }
});

mManager.stopPeerDiscovery(mChannel, new WifiP2pManager.ActionListener() {
    @Override
    public void onSuccess() {
        Log.d(TAG, "onSuccess: stopSearchDevice");
    }

    @Override
    public void onFailure(int reason) {
        Log.d(TAG, "onFailure: stopSearchDevice" + reason);
    }
});
````

在调用`discoverPeers()`方法后，我们还获取不到任何对等设备的信息。此时若成功检测到对等设备，则系统会发送`WIFI_P2P_PEERS_CHANGED_ACTION`广播，此时我们通过`requestPeers()`方法即可获取到对等设备的列表，详见上述`WiFiDirectReceiver`代码。

#### 设置WifiP2pWfdInfo

这是Miracast服务发现中最重要的一步，我们需要调用`setWFDInfo()`方法启动WFD，这样在发送端才能搜索到此设备。
其中`WifiP2pWfdInfo`类与`setWFDInfo()`方法都是`@hide`的，在标准的sdk中无法直接进行访问。

针对`WifiP2pWfdInfo`类，因为它没有太多依赖，我们可以直接从源码中拷贝一份出来，放在`android.net.wifi.p2p`包下：
```java
/**
 * A class representing Wifi Display information for a device
 * @hide
 */
public class WifiP2pWfdInfo implements Parcelable {
	...
    public WifiP2pWfdInfo() {
    }

    public WifiP2pWfdInfo(int devInfo, int ctrlPort, int maxTput) {
        mWfdEnabled = true;
        mDeviceInfo = devInfo;
        mCtrlPort = ctrlPort;
        mMaxThroughput = maxTput;
    }
    ...
}
```

对于`setWFDInfo()`方法，我们可以采用反射的方式进行调用：
```java
private void setWFDInfoInner(WifiP2pWfdInfo wfdInfo) {
    try {
        Method method = ReflectUtil.getPrivateMethod(mManager.getClass(), "setWFDInfo",
                WifiP2pManager.Channel.class,
                WifiP2pWfdInfo.class,
                WifiP2pManager.ActionListener.class);
        method.invoke(mManager, mChannel, wfdInfo, new WifiP2pManager.ActionListener() {
            @Override
            public void onSuccess() {
                Log.d(TAG, "setWFDInfo onSuccess:");
            }

            @Override
            public void onFailure(int reason) {
                Log.e(TAG, "setWFDInfo onFailure:" + reason);
            }
        });
    } catch (IllegalAccessException e) {
        e.printStackTrace();
    } catch (InvocationTargetException e) {
        e.printStackTrace();
    }
}
```

初始化`WifiP2pWfdInfo`，并进行如下设置，重要的几个字段详见注释：
```java
public void setWfdInfo() {
    final WifiP2pWfdInfo wfdInfo = new WifiP2pWfdInfo();
    // 开启WiFi Display
    wfdInfo.setWfdEnabled(true);
    wfdInfo.setSessionAvailable(true);
    wfdInfo.setCoupledSinkSupportAtSink(false);
    wfdInfo.setCoupledSinkSupportAtSource(false);
    // 设置设备模式为SINK端（Miracast接收端）
    wfdInfo.setDeviceType(WifiP2pWfdInfo.PRIMARY_SINK);
    wfdInfo.setControlPort(WFD_DEFAULT_PORT);
    wfdInfo.setMaxThroughput(WFD_MAX_THROUGHPUT);
    setWFDInfoInner(wfdInfo);
}
```

若希望关闭WiFi Display模式，则直接`setWfdEnabled(false)`即可：
```java
public void clearWfdInfo() {
    final WifiP2pWfdInfo wfdInfo = new WifiP2pWfdInfo();
    wfdInfo.setWfdEnabled(false);
    setWFDInfoInner(wfdInfo);
}
```

完成上述步骤后，在发送端的投屏或者投射功能中，应该能搜索到对应的Miracast设备了。

### Wi-Fi P2P 连接

在发送端搜索到Miracast设备，并点击对应设备后，就进入到了连接过程。此时Sink端应该会弹出一个[连接邀请]的授权窗口，可以选择拒绝或者接受。选择接受后，若是第一次连接，则会进入到GO协商的过程。

#### GO协商（Group Owner Negotiation）
GO协商是一个复杂的过程，共包含三个类型的Action帧：GO Req、GO Resp、GO Confirm，经过这几个帧的交互最终确认是Sink端还是Source端作为Group Owner，因此谁做GO是不确定的。那具体的协商规则是怎样的呢？官方的流程图清晰地给出了答案：
![](go-determination-flowchart.jpg)

首先通过`Group Owner Intent`的值进行协商，值大者为GO。若Intent值相同就需要判断Req帧中`Tie breaker`位，置1者为GO。若2台设备都设置了Intent为最大值，都希望能成为GO，则这次协商失败。

那么，如何设置这个Intent值呢？发送端在`connect()`的时候，可通过`groupOwnerIntent`字段设置GO的优先级的（范围从0-15，0表示最小优先级），方法如下：
```java
WifiP2pConfig config = new WifiP2pConfig();
...
config.groupOwnerIntent = 15; // I want this device to become the owner
mManager.connect(mChannel, config, actionListener);
```
>PS: 对GO完整协商过程感兴趣的童鞋可以查看`Wi-Fi P2P Technical Specification`文档的`3.1.4.2 Group Owner Negotiation`这章

Miracast Sink端的场景为接收端，因此不能通过`groupOwnerIntent`字段来设置GO优先级。那么还有其他方式可以让Sink端成为GO吗？毕竟在多台设备通过Miracast投屏的时候，Sink端是必须作为GO才能实现的。答案其实也很简单，就是自己创建一个组，自己成为GO，让其他Client加进来，在连接前直接调用`createGroup()`方法即可完成建组操作：
```java
mManager.createGroup(mChannel, new WifiP2pManager.ActionListener() {
    @Override
    public void onSuccess() {
        Log.d(TAG, "createGroup onSuccess");
    }

    @Override
    public void onFailure(int reason) {
        Log.d(TAG, "createGroup onFailure:" + reason);
    }
});
```

建组成功后我们可以通过`requestGroupInfo()`方法来查看组的基本信息，以及组内Client的情况：
```java
mManager.requestGroupInfo(mChannel, wifiP2pGroup -> {
    Log.d(TAG, "onGroupInfoAvailable detail:\n" + wifiP2pGroup.toString());
    Collection<WifiP2pDevice> clientList = wifiP2pGroup.getClientList();
    if (clientList != null) {
        int size = clientList.size();
        Log.d(TAG, "onGroupInfoAvailable - client count:" + size);
        // Handle all p2p client devices
    }
});
```

GO协商完毕，并且`Wi-Fi Direct`连接成功的时候，我们将会收到`WIFI_P2P_CONNECTION_CHANGED_ACTION`这个广播，此时我们可以调用 `requestConnectionInfo()`，并在`onConnectionInfoAvailable()`回调中通过`isGroupOwner`字段来判断当前设备是Group Owner，还是Peer。通过`groupOwnerAddress`，我们可以很方便的获取到Group Owner的IP地址。
```java
@Override
public void onConnectionInfoAvailable(WifiP2pInfo wifiP2pInfo) {
    if (wifiP2pInfo.groupFormed && wifiP2pInfo.isGroupOwner) {
        Log.d(TAG, "is groupOwner: ");
    } else if (wifiP2pInfo.groupFormed) {
        Log.d(TAG, "is peer: ");
    }
    String ownerIP = wifiP2pInfo.groupOwnerAddress.getHostAddress();
    Log.d(TAG, "onConnectionInfoAvailable ownerIP = " + ownerIP);
}
```

受WiFi P2P API的限制，各设备获取到的MAC和IP地址情况如下图所示：
![](groupOwner.jpg)

由于在后续RTSP进行指令通讯的时候，需要通过Socket与Source端建立连接，也就是我们需要先知道Source端的IP地址与端口。根据上图，我们可能出现以下2种情况：

- 情况1：Sink端为Peer，Source端为GO。

这种情况下，Sink端知道Source端（GO）的IP地址，可以直接进行Socket连接。

- 情况2：Sink端为GO，Source端为Peer。

这种情况下，Sink端只知道自己（GO）的IP地址，不知道Source端（Peer）的IP地址，但此时能获取到MAC地址。

#### 通过ARP协议获取对应MAC设备的IP地址

针对上述情况2，我们需要通过MAC地址获取到对应主机的IP地址，以完成与Source端的Socket连接，比较经典的方案是采用解析ARP缓存表的形式进行。

ARP（Address Resolution Protocol），即地址解析协议，是根据IP地址获取物理地址的一个TCP/IP协议。主机发送信息时将包含目标IP地址的ARP请求广播到局域网络上的所有主机，并接收返回消息，以此确定目标的物理地址；收到返回消息后将该IP地址和物理地址存入本机ARP缓存中并保留一定时间，下次请求时直接查询ARP缓存以节约资源。

在Android上，我们可以通过以下指令获取ARP缓存表：

- 方法1：通过busybox arp指令
```
dior:/ $ busybox arp
? (192.168.0.108) at f8:ff:c2:10:e7:62 [ether]  on wlan0
? (192.168.0.1) at 9c:a6:15:d6:e8:f4 [ether]  on wlan0
```

- 方法2：通过cat proc/net/arp命令
```
dior:/ $ cat proc/net/arp
IP address       HW type     Flags       HW address            Mask     Device
192.168.0.108    0x1         0x2         f8:ff:c2:10:e7:62     *        wlan0
192.168.0.1      0x1         0x2         9c:a6:15:d6:e8:f4     *        wlan0
```

剩下的工作就是采用强大的正则表达式解析返回的字符串，并查找出对应MAC设备的IP地址了。

#### 获取Source端RTSP端口号

经过上面的步骤，我们已经拿到了Source端的IP地址，只剩下端口号了。这一步就比较简单了，通过`requestPeers()`方法获取已连接的对等设备`WifiP2pDevice`，再获取其中的`WifiP2pWfdInfo`即可拿到端口号：
```java
 mManager.requestPeers(mChannel, peers -> {
    Collection<WifiP2pDevice> devices = peers.getDeviceList();
    for (WifiP2pDevice device : devices) {
        boolean isConnected = (WifiP2pDevice.CONNECTED == device.status);
        if (isConnected) {
            int port = getDevicePort(device);
            break;
        }
    }
});
```

这里由于`WifiP2pDevice`中的`wfdInfo`字段为`@hide`，因此需要通过反射的方式获取`WifiP2pWfdInfo`。最后通过`getControlPort()`方法即可拿到Source端RTSP端口号：
```java
public int getDevicePort(WifiP2pDevice device) {
    int port = WFD_DEFAULT_PORT;
    try {
        Field field = ReflectUtil.getPrivateField(device.getClass(), "wfdInfo");
        if (field == null) {
            return port;
        }
        WifiP2pWfdInfo wfdInfo = (WifiP2pWfdInfo) field.get(device);
        if (wfdInfo != null) {
            port = wfdInfo.getControlPort();
            if (port == 0) {
                Log.w(TAG,"set port to WFD_DEFAULT_PORT");
                port = WFD_DEFAULT_PORT;
            }
        }
    } catch (IllegalAccessException e) {
        e.printStackTrace();
    }
    return port;
}
```

拿到了Source端的IP地址与端口号后，我们就可以建立RTSP连接，建立后续控制指令的通道了，详见下篇博客。


### 参考
- https://developer.android.com/guide/topics/connectivity/wifip2p
- https://developer.android.com/training/connect-devices-wirelessly/wifi-direct