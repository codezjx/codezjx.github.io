---
title: Miracast技术详解（五）：Permission 问题处理
date: 2020-06-16 21:20:13
categories: Miracast
tags: [Android, Miracast, Wi-Fi Display, Permission]
---

## Permission 问题处理

由于Android上的Miracast功能强依赖Wi-Fi P2P，因此这个过程中也会依赖其相应的权限。经过调试及踩坑，主要会涉及到以下几个权限问题。
>以下分析过程中涉及到的源码版本为`android-8.1.0_r60`

### WFD Permission
自Android 8.0及以后，官方已经限制了对`setWFDInfo()`接口的调用（这个接口本来也是@hide的，因此官方在高版本中对其进行限制也是理所当然），普通app已经没有权限进行调用了，也就是第三方app已经不能实现Sink端了。所以市面上的一些投屏软件，如：AirScreen，在高版本中会弹窗提示功能已被Google禁用。除非你是系统应用或者有系统签名才能突破此限制，此时你会收到类似如下的报错：`Wifi Display Permission denied for uid = 10104`
```java
W/System.err: java.lang.reflect.InvocationTargetException
W/System.err:     at java.lang.reflect.Method.invoke(Native Method)
W/System.err:     at com.codezjx.miracastsdk.WfdManager.setWFDInfoInner(WfdManager.java:143)
W/System.err:     at com.codezjx.miracastsdk.WfdManager.setWfdInfo(WfdManager.java:106)
W/System.err:     at com.codezjx.miracastsdk.WfdManager.startSearch(WfdManager.java:207)
W/System.err:     at java.util.TimerThread.mainLoop(Timer.java:555)
W/System.err:     at java.util.TimerThread.run(Timer.java:505)
W/System.err: Caused by: java.lang.SecurityException: Wifi Display Permission denied for uid = 10104
W/System.err:     at android.os.Parcel.readException(Parcel.java:2013)
W/System.err:     at android.os.Parcel.readException(Parcel.java:1959)
W/System.err:     at android.net.wifi.p2p.IWifiP2pManager$Stub$Proxy.checkConfigureWifiDisplayPermission(IWifiP2pManager.java:201)
W/System.err:     at android.net.wifi.p2p.WifiP2pManager.setWFDInfo(WifiP2pManager.java:1379)
W/System.err:     ... 7 more
```

我们来跟下源码，看下官方究竟做了什么，首先直接查看`WifiP2pManager`的`setWFDInfo()`方法。我们可以看到在`sendMessage()`发送通知前，增加了`checkConfigureWifiDisplayPermission()`的权限校验：
```java
/** @hide */
public void setWFDInfo(
        Channel c, WifiP2pWfdInfo wfdInfo,
        ActionListener listener) {
    checkChannel(c);
    try {
        mService.checkConfigureWifiDisplayPermission();
    } catch (RemoteException e) {
        e.rethrowFromSystemServer();
    }
    c.mAsyncChannel.sendMessage(SET_WFD_INFO, 0, c.putListener(listener), wfdInfo);
}
```

其中`mService`为`IWifiP2pManager`类型的Binder接口，通过IPC的形式最终调用到`WifiP2pServiceImpl`中，我们直接查看该方法实现。可以看到增加了对`android.Manifest.permission.CONFIGURE_WIFI_DISPLAY`权限的校验，而此权限仅向系统应用开放。
```java
public class WifiP2pServiceImpl extends IWifiP2pManager.Stub {
    ...
    @Override
    public void checkConfigureWifiDisplayPermission() {
        if (!getWfdPermission(Binder.getCallingUid())) {
            throw new SecurityException("Wifi Display Permission denied for uid = "
                    + Binder.getCallingUid());
        }
    }

    private boolean getWfdPermission(int uid) {
        if (mWifiInjector == null) {
            mWifiInjector = WifiInjector.getInstance();
        }
        WifiPermissionsWrapper wifiPermissionsWrapper = mWifiInjector.getWifiPermissionsWrapper();
        return wifiPermissionsWrapper.getUidPermission(
                android.Manifest.permission.CONFIGURE_WIFI_DISPLAY, uid)
                != PackageManager.PERMISSION_DENIED;
    }
    ...
}
```

因此在Android 8.0之后，第三方应用程序已经无法实现Sink接收端。如果你是厂商App，应用有系统权限，则可以绕过此限制。或者找厂商修改frameworks源码，增加app白名单开放此权限，也可以绕过，具体可在`getWfdPermission()`方法中增加过滤判断，然后返回true即可。

### Denied: no location permission
在绕过了上述的WFD Permission权限之后，你可能还会遇到坑。具体的现象就是，调用`WifiP2pManager`的`requestPeers()`或者`requestGroupInfo()`方法的时候，可能会返回空的peers列表。毕竟在Sink与Source端建好组后，是需要通过这些接口来获取P2P对等设备，进而建立RTSP连接的。如果你忘记了这几个函数的用法与场景，可以回看[《Miracast技术详解（一）：Wi-Fi Display》][1]这篇文章。刚开始遇到这个问题你可能会一脸懵逼，但是细心的同学可能会发现Logcat中会打印这么一句（不要过滤当前进程log，要查看全局log）：
```java
D/WifiPermissionsUtil: Denied: no location permission
```

我们可以从这句Logcat报错作为一个源头，查找为何没有生效。正常情况下，按照文章[《Miracast技术详解（一）：Wi-Fi Display》][1]的操作方法，应该能完整的实现Sink端的设备发现及RTSP连接的准备工作了。我们首先从上述调用异常的方法`requestPeers()`源码看起：
```java
/**
 * Request the current list of peers.
 *
 * @param c is the channel created at {@link #initialize}
 * @param listener for callback when peer list is available. Can be null.
 */
public void requestPeers(Channel c, PeerListListener listener) {
    checkChannel(c);
    Bundle callingPackage = new Bundle();
    callingPackage.putString(CALLING_PACKAGE, c.mContext.getOpPackageName());
    c.mAsyncChannel.sendMessage(REQUEST_PEERS, 0, c.putListener(listener),
            callingPackage);
}
```

其中通过`sendMessage()`发送通知，并最终回调到`WifiP2pServiceImpl`的`P2pStateMachine.DefaultState`中进行处理：
```java
...
case WifiP2pManager.REQUEST_PEERS:
    replyToMessage(message, WifiP2pManager.RESPONSE_PEERS,
            getPeers((Bundle) message.obj, message.sendingUid));
    break;
...
```

我们继续查看`getPeers()`方法的内部实现，终于看到了与Logcat中报错相关的`WifiPermissionsUtil`方法调用，由此可以猜测，应该是某些权限的校验失败了，导致获取不了扫描结果。
```java
/**
 * Enforces permissions on the caller who is requesting for P2p Peers
 * @param pkg Bundle containing the calling package string
 * @param uid of the caller
 * @return WifiP2pDeviceList the peer list
 */
private WifiP2pDeviceList getPeers(Bundle pkg, int uid) {
    String pkgName = pkg.getString(WifiP2pManager.CALLING_PACKAGE);
    boolean scanPermission = false;
    WifiPermissionsUtil wifiPermissionsUtil;
    // getPeers() is guaranteed to be invoked after Wifi Service is up
    // This ensures getInstance() will return a non-null object now
    if (mWifiInjector == null) {
        mWifiInjector = WifiInjector.getInstance();
    }
    wifiPermissionsUtil = mWifiInjector.getWifiPermissionsUtil();
    // Minimum Version to enforce location permission is O or later
    try {
        scanPermission = wifiPermissionsUtil.canAccessScanResults(pkgName, uid,
                Build.VERSION_CODES.O);
    } catch (SecurityException e) {
        Log.e(TAG, "Security Exception, cannot access peer list");
    }
    if (scanPermission) {
        return new WifiP2pDeviceList(mPeers);
    } else {
        return new WifiP2pDeviceList();
    }
}
```

继续深入`canAccessScanResults()`方法内部源码，查看到底哪里出问题，此时终于找到了与Logcat中吻合的异常输出`Denied: no location permission`，由此可以判断`canCallingUidAccessLocation`和`canAppPackageUseLocation`属性都为false导致。由于涉及case过多，为了提高排查效率，我们可以打印出相关的boolean变量进行排查。有条件的同学可以通过修改frameworks源码增加log打印的方式进行，或通过一些hook框架，获取上面相关变量的值。
```java
/**
 * API to determine if the caller has permissions to get
 * scan results.
 * @param pkgName package name of the application requesting access
 * @param uid The uid of the package
 * @param minVersion Minimum app API Version number to enforce location permission
 * @return boolean true or false if permissions is granted
 */
public boolean canAccessScanResults(String pkgName, int uid,
            int minVersion) throws SecurityException {
    mAppOps.checkPackage(uid, pkgName);
    // Check if the calling Uid has CAN_READ_PEER_MAC_ADDRESS
    // permission or is an Active Nw scorer.
    boolean canCallingUidAccessLocation = checkCallerHasPeersMacAddressPermission(uid)
            || isCallerActiveNwScorer(uid);
    // LocationAccess by App: For AppVersion older than minVersion,
    // it is sufficient to check if the App is foreground.
    // Otherwise, Location Mode must be enabled and caller must have
    // Coarse Location permission to have access to location information.
    boolean canAppPackageUseLocation = isLegacyForeground(pkgName, minVersion)
            || (isLocationModeEnabled(pkgName)
                    && checkCallersLocationPermission(pkgName, uid));
    // If neither caller or app has location access, there is no need to check
    // any other permissions. Deny access to scan results.
    if (!canCallingUidAccessLocation && !canAppPackageUseLocation) {
        mLog.tC("Denied: no location permission");
        return false;
    }
    // Check if Wifi Scan request is an operation allowed for this App.
    if (!isScanAllowedbyApps(pkgName, uid)) {
        mLog.tC("Denied: app wifi scan not allowed");
        return false;
    }
    // If the User or profile is current, permission is granted
    // Otherwise, uid must have INTERACT_ACROSS_USERS_FULL permission.
    if (!canAccessUserProfile(uid)) {
        mLog.tC("Denied: Profile not permitted");
        return false;
    }
    return true;
}
```

首先查看`checkCallerHasPeersMacAddressPermission()`方法，由于此权限`PEERS_MAC_ADDRESS`仅授权给系统应用，因此一般第三方app这里直接就返回false了。
```java
/**
 * Returns true if the caller holds PEERS_MAC_ADDRESS permission.
 */
private boolean checkCallerHasPeersMacAddressPermission(int uid) {
    return mWifiPermissionsWrapper.getUidPermission(
            android.Manifest.permission.PEERS_MAC_ADDRESS, uid)
            == PackageManager.PERMISSION_GRANTED;
}
```

然后是`isCallerActiveNwScorer()`方法，看注释应该是判断调用方app是否是“网络评分器”(google翻译过来)，感觉一般app也是会返回false。因此`canCallingUidAccessLocation`这个case，应该就是false了。
```java
/**
 * Returns true if the caller is an Active Network Scorer.
 */
private boolean isCallerActiveNwScorer(int uid) {
    return mNetworkScoreManager.isCallerActiveScorer(uid);
}
```

我们继续来分析`isLegacyForeground()`方法，这里在上层`getPeers()`传入的version为`Build.VERSION_CODES.O`，由于我们程序的`targetSdkVersion`大于26，因此这个条件的值也是false了。
```java
private boolean isLegacyForeground(String pkgName, int version) {
    return isLegacyVersion(pkgName, version) && isForegroundApp(pkgName);
}
/**
 * Returns true if the App version is older than minVersion.
 */
private boolean isLegacyVersion(String pkgName, int minVersion) {
    try {
        if (mContext.getPackageManager().getApplicationInfo(pkgName, 0)
                .targetSdkVersion < minVersion) {
            return true;
        }
    } catch (PackageManager.NameNotFoundException e) {
        // In case of exception, assume known app (more strict checking)
        // Note: This case will never happen since checkPackage is
        // called to verify valididity before checking App's version.
    }
    return false;
}
```

分析到这里，唯一的答案只能是`isLocationModeEnabled()`与`checkCallersLocationPermission()`同时为true，变量`canAppPackageUseLocation`才可能为true，才不会走到报错的逻辑中。因此，在程序运行的过程中，两者其一为false都将导致`Denied: no location permission`的报错。接下来，我们来详细分析下这两个case，如何才能保证其值为true。

### LocationMode

关于LocationMode，在Android设置中，我们可以找到“位置信息”这个服务。如果将此模式关闭，则方法返回`Settings.Secure.LOCATION_MODE_OFF`。开启的情况下，有可能为以下几个值，分别对应：仅限设备、低耗电量和高精确度：
```java
/**
 * Network Location Provider disabled, but GPS and other sensors enabled.
 */
public static final int LOCATION_MODE_SENSORS_ONLY = 1;
/**
 * Reduced power usage, such as limiting the number of GPS updates per hour. Requests
 * with {@link android.location.Criteria#POWER_HIGH} may be downgraded to
 * {@link android.location.Criteria#POWER_MEDIUM}.
 */
public static final int LOCATION_MODE_BATTERY_SAVING = 2;
/**
 * Best-effort location computation allowed.
 */
public static final int LOCATION_MODE_HIGH_ACCURACY = 3;
```

![](location-mode.png)

我们可以通过`Settings.Secure.getInt()`来获取LocationMode的开启状态，以及当前所处的模式，详见以下代码：

```java
private boolean isLocationModeEnabled(String pkgName) {
    // Location mode check on applications that are later than version.
    return (mSettingsStore.getLocationModeSetting(mContext)
             != Settings.Secure.LOCATION_MODE_OFF);
}
/**
 * Get Location Mode settings for the context
 * @param context
 * @return Location Mode setting
 */
public int getLocationModeSetting(Context context) {
    return Settings.Secure.getInt(context.getContentResolver(),
          Settings.Secure.LOCATION_MODE, Settings.Secure.LOCATION_MODE_OFF);
}
```

因此，这里我们可以得出一个结论，若系统关闭了LocationMode，则`isLocationModeEnabled()`返回值为false，上层调用会失效，报`Denied: no location permission`错误，这里我们需要友好的提示用户手动进行开启。

### ACCESS_FINE_LOCATION Permission

还记得在文章[《Miracast技术详解（一）：Wi-Fi Display》][1]中，我们已经在Manifest中按照官方实现声明了`ACCESS_FINE_LOCATION`权限，关于`ACCESS_COARSE_LOCATION`与它的关系，只要授权了`ACCESS_FINE_LOCATION`权限，则默认包含了后者，可理解为包含的关系，详见[stackoverflow][2]上的这个解答。那`checkCallersLocationPermission()`这个case的意图就很明显了，就是检测我们的app是否进行了`ACCESS_FINE_LOCATION`的授权。

```java
/**
 * Checks that calling process has android.Manifest.permission.ACCESS_COARSE_LOCATION
 * and a corresponding app op is allowed for this package and uid.
 *
 * @param pkgName PackageName of the application requesting access
 * @param uid The uid of the package
 */
public boolean checkCallersLocationPermission(String pkgName, int uid) {
    // Coarse Permission implies Fine permission
    if ((mWifiPermissionsWrapper.getUidPermission(
            Manifest.permission.ACCESS_COARSE_LOCATION, uid)
            == PackageManager.PERMISSION_GRANTED)
            && checkAppOpAllowed(AppOpsManager.OP_COARSE_LOCATION, pkgName, uid)) {
        return true;
    }
    return false;
}
```

这里关键，是在程序中要处理好动态授权与授权异常的提示，若用户不授予`ACCESS_FINE_LOCATION`权限，上层逻辑是会受到极大影响的，而且这个报错还不那么明显。通过`checkSelfPermission()`及`requestPermissions()`方法，我们可以很简单就能完成权限检测及动态授权，详见以下示例：
```java
private boolean checkPermissions() {
    // For wifi p2p requestPeers permission
    int checkResult = ContextCompat.checkSelfPermission(getApplicationContext(),
            Manifest.permission.ACCESS_FINE_LOCATION);
    if (checkResult != PackageManager.PERMISSION_GRANTED) {
        ActivityCompat.requestPermissions(this, new String[]{Manifest.permission.ACCESS_FINE_LOCATION}, REQUEST_CODE);
        return false;
    }
    return true;
}

@Override
public void onRequestPermissionsResult(int requestCode, @NotNull String[] permissions, @NotNull int[] grantResults) {
    if (requestCode != REQUEST_CODE) {
        return;
    }
    if (grantResults[0] != PackageManager.PERMISSION_GRANTED) {
        String msg = "Permission not granted: " + permissions[0];
        Toast.makeText(this, msg, Toast.LENGTH_LONG).show();
    }
    ...
}
```

在最终确保LocationMode处于开启以及动态授权`ACCESS_FINE_LOCATION`成功后，变量`canAppPackageUseLocation`才可能为true，此时才不会报`Denied: no location permission`错误，上层调用`WifiP2pManager`的`requestPeers()`或者`requestGroupInfo()`方法才能正常返回结果。写到这里，基本上实现Sink端会遇到的权限问题都讲解完了，希望大家不要再踩坑哈。

[1]: https://codezjx.com/posts/miracast-wifi-display/
[2]: https://stackoverflow.com/questions/15310742/if-i-have-access-fine-location-already-can-i-omit-access-coarse-location