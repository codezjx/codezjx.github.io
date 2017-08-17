---
title: Android中Recent键及状态栏屏蔽
date: 2015-05-06 22:24:00
categories: Android
tags: [Android, Hack]
---

Back键和Home键的屏蔽可以看我这篇贴：
https://codezjx.github.io/2015/05/04/disable-back-and-home-key/

## Recent键及状态栏下拉的屏蔽
Home键与Recent键的点击事件是在framework层进行处理的，因此`onKeyDown`与`dispatchKeyEvent`都捕获不到点击事件。
查看`StatusBarManager.java`源码，目前只能通过其`void disable(int what) {…}`设置，并可传入值：
```java
public static final int DISABLE_HOME = View.STATUS_BAR_DISABLE_HOME;
public static final int DISABLE_RECENT = View.STATUS_BAR_DISABLE_RECENT;
public static final int DISABLE_BACK = View.STATUS_BAR_DISABLE_BACK;
public static final int DISABLE_NONE = 0x00000000;
```
等等…    具体值可以查view的源码，因为都是`@hide`的
首先想到了反射机制来进行调用：（或者使用移除了@hide的API 库classes.jar）
```java
public static final String STATUS_BAR_SERVICE = “statusbar”;
public static final String CLASS_STATUS_BAR_MANAGER = “android.app.StatusBarManager”;
public static final String METHOD_DISABLE = “disable”;
try {
    Object service = getSystemService(STATUS_BAR_SERVICE);
    Class <?> statusBarManager = Class.forName(CLASS_STATUS_BAR_MANAGER);
    Method disable = statusBarManager.getMethod(METHOD_DISABLE, int.class);
    disable.invoke (service, 0x01000000); //为View.STATUS_BAR_DISABLE_RECENT的值
} catch (Exception e) {
    e.printStackTrace();
}
```

会报出以下错误提示：
> Neither user 10076 nor current process has android.permission.STATUS_BAR.

提示缺少权限，Manifest添加之，提示：`Permission is only granted to system apps`
**总结：通过这种方法屏蔽状态栏下拉，必须得有系统签名，WTF。。。**

**后续：**参考了一些锁屏软件，如小米百变锁屏、GO锁屏等，他们在锁屏的时候也没有屏蔽掉Recent键和状态栏下拉。而是在点击按键后迅速将弹出的Dialog和Window收回去，其实这也可以看作另一种应急解决方法。继续回去看`StatusBarManager.java`源码：发现了另外一个方法，`public void collapse() {…}`和`public void expand() {…}`，其中collapse为收缩的方法，expand为展开方法，而且调用这两个方法不需要系统签名。用上面的反射方法继续调用之：
注意：点击Recent键弹出dialog会回调`onWindowFocusChanged()`，因此可在该方法中调用：
```java
public void onWindowFocusChanged(boolean hasFocus) {
    super.onWindowFocusChanged(hasFocus);
    if(!hasFocus) {
        ……
        Method collapse = statusBarManager.getMethod(METHOD_COLLAPSE);
        collapse.invoke(service);
        ……
    }
}
```
加入权限：
```xml
<uses-permission android:name=”android.permission.EXPAND_STATUS_BAR”/>
```
后面又发现了另一种方法，更加简便，同样是点击后迅速消失（还可以隐藏长按电源的弹出框）：
```java
sendBroadcast(new Intent(Intent.ACTION_CLOSE_SYSTEM_DIALOGS));
```
可以通过View焦点丢失的情况下去触发，发送该广播，如：
```java
class MyView extends View {

    public MyView(Context pContext) {
        super(pContext);
    }

    @Override
    public void onWindowFocusChanged(boolean pHasWindowFocus) {
        super.onWindowFocusChanged(pHasWindowFocus);
        if (!pHasWindowFocus) {
            mContext.sendBroadcast(new Intent(Intent.ACTION_CLOSE_SYSTEM_DIALOGS));
        }
    }
}
```


## 状态栏的隐藏

### 方法一
隐藏：（但点击menu会恢复，然后又自动隐藏）
```java
getWindow().getDecorView().setSystemUiVisibility(Window.FEATURE_ACTION_BAR_OVERLAY);
```
显示：
```java
getWindow().getDecorView().setSystemUiVisibility(View.SYSTEM_UI_FLAG_VISIBLE);
```
（用这个type可以将按钮变成小圆点：`View.SYSTEM_UI_FLAG_LOW_PROFILE`）
```java
WindowManager.LayoutParams params = window.getAttributes();
params.systemUiVisibility = View.SYSTEM_UI_FLAG_LOW_PROFILE;
window.setAttributes(params);
```

### 方法二
隐藏：进入`\system\`目录，编辑`build.prop`，在最后一行添加`qemu.hw.mainkeys=1`，保存重启即可。
显示：改成`qemu.hw.mainkeys=0`，保存重启即可。

### 方法三
参考一个叫GMD Hide Bar的软件。反编译了他的源码，他是用命令行直接干掉SystemUI这个进程来达到隐藏目的的。大家可以尝试直接在DDMS中关闭：`com.android.systemui`这个进程，状态栏直接消失，不过一段时间后又会自动运行。因此需要运行后台线程不断执行关闭操作，比较耗资源，且需要root权限。
隐藏：
```java
int k = getPid();
if (k > 0)
ProcessUtil.execAsRoot(“kill -s CONT ” + k);
```
显示：
```java
ProcessUtil.execAsApp(“am startservice -n com.android.systemui/.SystemUIService”);
```






