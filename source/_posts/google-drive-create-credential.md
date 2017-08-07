---
title: Google Drive开启API和创建Credential
date: 2016-03-31 13:05:06
categories: Google Drive Android
tags: [Android, Google Drive]
---

## 开启Drive API和创建Credential

- 首先按照[官网流程][1]在Google Developers Console创建好Project，并开启Drive API。
![这里写图片描述](20160331130158970.png)

- 然后进入Credentials界面新建一个OAuth 2.0 client凭证。
![这里写图片描述](20160331130246189.png)

- Application type里面会有以下几种类型：
    - Web application
    - Android
    - Chrome App
    - iOS
    - PlayStation 4
    - Other

    根据自己的需求，若是在Android App上配合原生Drive SDK使用，可以选择Android（需要fingerprint和packageName）；若是采用Web的接口进行授权及验证：例如java控制台程序，webview的方式等，则可以选择Web application或者Other。因为我后续打算使用Web的接口进行授权，所以此处直接选择Other。新建好以后将拿到Client ID和Client secret，他们主要用于后续的Oauth2.0认证以及API请求。


## Drive SDK for Android
Google官方提供了专门针对Android的一套[Drive SDK][2]，优缺点如下：

- 优点
1.方便、快捷
2.提供完整的API接口，客户端代码量极少
3.与Google Account账号体系结合，不需要自己管理账号认证等等

- 缺点
没有安装Google Play Service将无法工作


相信大家看到上面这条缺点后就知道在咱们大天朝是不可能用上Drive SDK了，就算解决了肉身翻墙的问题，Google Service全家桶估计没有几个用户的手机是预装的。所以，我们只能采用另外一种方式来进行Google Oauth2.0的登录、认证、授权问题。


[1]: https://developers.google.com/drive/v3/web/quickstart/java
[2]: https://developers.google.com/drive/android/