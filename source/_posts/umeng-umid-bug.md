---
title: 友盟统计UMID潜在的一个坑
date: 2016-07-04 21:09:01
tags: [Umeng, 友盟, SDK, Bug]
---

最近发现友盟的数据统计里面，活跃用户的数量有点不大对劲，跟启动次数相比，严重偏少。sdk的使用方式没啥好说的，就那么简单几步，应该不会是sdk设置的问题。于是就从友盟关于活跃用户的定义开始，着手分析这个问题。

关于活跃用户的定义，可以参考官方这篇文章：[你真的了解活跃用户吗？][1]，总结起来其实就是很简单的一句：
> 活跃用户的定义：打开应用的用户即为活跃用户，不考虑用户的使用情况。

从上面的文章，了解到Umeng里面对用户的定义：友盟将一个独立的设备视为一个用户，然而每个独立的用户是通过UMID来进行唯一标识的。然而UMID又是神马鬼东西？详见这篇文章：[友盟UMID方案解析][2]，简单来说就是友盟会在第一次安装的时候生成一个UMID，当ID生成以后友盟会尽量保证这个UMID不会发生变化。

在应用对应的存储目录下面，我们可以找到这个UMID的身影：
```
/data/data/com.xxxx.application/files/.imprint
/data/data/com.xxxx.application/files/.umeng/exchangeIdentity.json
```

拿`exchangeIdentity.json`这个文件来说，cat一下里面的内容，应该可以看到：
```
{"appkey":"xxxxxxxxxxxxxxxxxxxxxxxx","umid":"528c8e6cd4a3c6598999a0e9df15ad32","signature":"xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx","checksum":"xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"}
```

这里笔者发现公司里多台设备的UMID都居然是一个相同的UMID值，WTF！！！也同样是上面这串神秘的代码：`528c8e6cd4a3c6598999a0e9df15ad32`。

这个时候就需要查一下UMID的生成方式了，从上面那篇UMID方案解析的文章中，可以了解到Android系统中与UMID相关的几个ID：imei、mac地址、android_id。有了这些关键点，我们就可以开始去反编译友盟的sdk包并进行下一步的搜索了（这里反编译了友盟最新的jar包：`umeng-analytics-v6.0.1.jar`）。。。果然，使用上面这几个关键字，很快就搜索到了一些关键的代码：
```java
  private static String I(Context paramContext)
  {
    String str = "";
    TelephonyManager localTelephonyManager = (TelephonyManager)paramContext.getSystemService("phone");
    if (Build.VERSION.SDK_INT >= 23)
    {
        // 此处省略一部分代码
    }
    else
    {
      if (localTelephonyManager != null) {
        try
        {
          if (a(paramContext, "android.permission.READ_PHONE_STATE")) {
            str = localTelephonyManager.getDeviceId();
          }
        }
        catch (Exception localException2)
        {
          bl.d(a, "No IMEI.", localException2);
        }
      }
      if (TextUtils.isEmpty(str))
      {
        bl.e(a, new Object[] { "No IMEI." });
        str = u(paramContext);
        if (TextUtils.isEmpty(str))
        {
          bl.e(a, new Object[] { "Failed to take mac as IMEI. Try to use Secure.ANDROID_ID instead." });
          str = Settings.Secure.getString(paramContext.getContentResolver(), "android_id");
          bl.a(a, new Object[] { "getDeviceId: Secure.ANDROID_ID: " + str });
        }
      }
    }
    return str;
  }
```

这段代码逻辑比较简单（由于笔者所调试系统 &lt; 23，故省略了一部分代码），首先`TelephonyManager.getDeviceId()`获取imei，若取不到则调用`u(context)`函数获取下一个字符串，若再取不到，则获取android_id。其实这里可以猜测到，u()中返回的字符串应该就是mac地址，我们来看下函数u()的逻辑代码：
```java
  public static String u(Context paramContext)
  {
    if (Build.VERSION.SDK_INT >= 23)
    {
      String str = d();
      if (str == null) {
        str = H(paramContext);
      }
      return str;
    }
    return H(paramContext);
  }
  
  private static String H(Context paramContext)
  {
    try
    {
      WifiManager localWifiManager = (WifiManager)paramContext.getSystemService("wifi");
      if (a(paramContext, "android.permission.ACCESS_WIFI_STATE"))
      {
        WifiInfo localWifiInfo = localWifiManager.getConnectionInfo();
        return localWifiInfo.getMacAddress();
      }
      bl.e(a, new Object[] { "Could not get mac address.[no permission android.permission.ACCESS_WIFI_STATE" });
      return "";
    }
    catch (Exception localException)
    {
      bl.e(a, new Object[] { "Could not get mac address." + localException.toString() });
    }
    return "";
  }
```

果然，函数`u(context)`就是返回wifi的mac地址的。那么，回到刚刚的那个问题，到底那串神秘的UMID是`528c8e6cd4a3c6598999a0e9df15ad32`根据啥来生成的？看着这格式有点像md5。然后把机器上的imei、mac地址、android_id都打印了出来：
```
07-01 18:01:05.586 16217-16217/com.xxxx.application D/xxx: imei: null
07-01 18:01:05.606 16217-16217/com.xxxx.application D/xxx: macAddr: 00:00:00:00:00:00
07-01 18:01:05.616 16217-16217/com.xxxx.application D/xxx: Secure.ANDROID_ID: 137xxxxxxxxxx16
```

突然发现公司设备上打印出来的mac地址都是00:00:00:00:00:00（因为木有wifi模块，只有ethernet模块，囧！！！），怒将其转为md5，正是上面的串代码。
```
字符串: 00:00:00:00:00:00
16位: d4a3c6598999a0e9
32位: 528c8e6cd4a3c6598999a0e9df15ad32
```

可是，为啥当mac地址是00:00:00:00:00:00的时候，不去选择android_id呢？，再回去仔细看代码，发现友盟用的是坑爹的`TextUtils.isEmpty()`来判断mac地址的有效性，跪了，上面那串明明就是无效的mac地址好么？只能说代码写得不严谨。。。
```java
  if (TextUtils.isEmpty(str))
  {
    bl.e(a, new Object[] { "No IMEI." });
    str = u(paramContext);
    if (TextUtils.isEmpty(str))
    {
      bl.e(a, new Object[] { "Failed to take mac as IMEI. Try to use Secure.ANDROID_ID instead." });
      str = Settings.Secure.getString(paramContext.getContentResolver(), "android_id");
      bl.a(a, new Object[] { "getDeviceId: Secure.ANDROID_ID: " + str });
    }
  }
```

至此，代码及原因分析完毕。当一些Android平板设备统一返回相同的mac地址，如00:00:00:00:00:00时（有可能是没有wifi模块；也有可能是山寨机出现这种情况的时候），友盟将会将其数据识别成同一用户，并且将会造成严重的MAC地址漂移。

[1]:  http://bbs.umeng.com/thread-6267-1-1.html
[2]:  http://bbs.umeng.com/thread-5850-1-1.html