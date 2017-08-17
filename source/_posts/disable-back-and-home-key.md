---
title: Android中Back键和Home键的屏蔽
date: 2015-05-04 21:16:00
categories: Android
tags: [Android, Hack]
---

## 屏蔽返回键
比较简单，重写`onBackPressed()`即可，不调用超类方法
```java
@Override
public void onBackPressed() {
    // super.onBackPressed();
}
```

## 屏蔽Home键
### 常规方法
代码如下，但是在Android4.0以上会失效
```java
@Override
public void onAttachedToWindow(){
    this.getWindow().setType(WindowManager.LayoutParams.TYPE_KEYGUARD);
    super.onAttachedToWindow();
}
```
并加入权限：
```xml
<uses-permission android:name=”android.permission.DISABLE_KEYGUARD”></uses-permission>
```

### Android4.0以上的屏蔽方法
此法较为猥琐,但在Android4.4以上会失效
用`WindowManager`的`addview`方法将view加到窗口上，加上的时候将view的`layoutparamas`的type设为`LayoutParams.TYPE_SYSTEM_ERROR`。

并加上权限
```xml
<uses-permission android:name=”android.permission.SYSTEM_ALERT_WINDOW”/>
```

原理：使用`WindowManager`在屏幕最前面加上一层view，并让其type设置为：`LayoutParams.TYPE_SYSTEM_ERROR`，官方对其解释是：**internal system error windows, appear on top of everything they can**，既显示在任何界面之上。并且设置flags为`LayoutParams.FLAG_NOT_TOUCHABLE`，这样我们后面一层的View才能监听到触摸事件。然后我们可以设置所add的view是一个空view，就不会感觉前面多了一层东西，从而达到屏蔽Home键的效果。

参考代码：
```java
private void forbiddenHomeKey(){
    mWindowManager = this.getWindowManager();
    mWindowManagerParams = new LayoutParams();
    mWindowManagerParams.width = LayoutParams.WRAP_CONTENT;
    mWindowManagerParams.height = LayoutParams.WRAP_CONTENT;
    // internal system error windows, appear on top of everything they can
    mWindowManagerParams.type = LayoutParams.TYPE_SYSTEM_ERROR;
     // indicate this view don’t respond the touch event
    mWindowManagerParams.flags = LayoutParams.FLAG_NOT_TOUCHABLE;
    // add an empty view on the top of the window
    mEmptyView = new View(this);
    mWindowManager.addView(mEmptyView, mWindowManagerParams);
}
```

为什么设置了这个type后就可以屏蔽Home呢？我们可以看看`PhoneWindowManager.java`的`interceptKeyBeforeDispatching()`方法：
```java
final int typeCount = WINDOW_TYPES_WHERE_HOME_DOESNT_WORK.length;
for (int i=0; i<typeCount; i++) {
    if (type == WINDOW_TYPES_WHERE_HOME_DOESNT_WORK[i]) {
        // don't do anything, but also don't pass it to the app
        return -1;
    }
}
```

`WINDOW_TYPES_WHERE_HOME_DOESNT_WORK`常量的值为：
```java
private static final int[] WINDOW_TYPES_WHERE_HOME_DOESNT_WORK = {
    WindowManager.LayoutParams.TYPE_SYSTEM_ALERT,
    WindowManager.LayoutParams.TYPE_SYSTEM_ERROR,
};
```
所以type设置为上面两个之一就可以了！

### 关于Home的屏蔽，还有一种思路
监听程序是否在前台显示（通过`ActivityManager.getRunningAppProcesses()`），如果没有，则马上把程序的task移动至前台（通过`ActivityManager.moveTaskToFront()`）。但是Android早就已经想到这个漏洞，当你点击完home键后，系统的Launcher会有5秒的延迟保护。所有启动Activity、或者移动到前台的方法都会有5秒延迟。具体看stackoverflow上的解答，若需要破解此限制需要加入系统权限`android.permission.STOP_APP_SWITCHES`：

http://stackoverflow.com/questions/5600084/starting-an-activity-from-a-service-after-home-button-pressed-without-the-5-seco

也就是说，若没有系统权限的话，只能通过自己写第三方的Launcher即可破解，此方法经过撸主本人亲测有效！
