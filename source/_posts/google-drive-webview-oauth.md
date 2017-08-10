---
title: Google Drive WebView授权方式实现
date: 2016-03-31 13:36:06
categories: Google Drive Android
tags: [Android, Google Drive, OAuth]
---

## 背景
由于国内大多数机器都木有安装Google Service框架，也就是说木有办法直接使用Drive SDK进行授权。因此，这一篇文章将介绍如何通过WebView的方式进行Oauth2.0授权码的获取（话说《ES文件浏览器》就是通过这种方式来实现Oauth2.0授权的）。具体的效果是这样的：如果没有登录Google账号，会要求你先登录一次
![这里写图片描述](20160331142025828.png)

## 先来看一下web端account的授权url：
```
https://accounts.google.com/o/oauth2/auth?
    redirect_uri=https://localhost/&
    response_type=code&
    client_id=xxxxxxxx-xxxxxxx.apps.googleusercontent.com&
    scope=https://www.googleapis.com/auth/drive+
             https://www.googleapis.com/auth/userinfo.profile
```

几个关键的字段我来解释一下：

 - redirect_uri：定义了授权码code如何发送到你指定的地址中（当用户点击了允许）
 - response_type：为固定值code，表示请求授权码
 - client_id：在上一篇文章中申请凭证的client id
 - scope：申请的权限范围（如上面的scope指定了获取用户drive最高权限以及获取用户信息权限，drive中具体的scope值可以参考[官网的说明][1]）

最后直接通过`webview.loadUrl(url)`方法就可以直接加载上面的页面了。


## 为WebView设置自定义WebViewClient
由于Oauth2.0中，验证的第一步就是需要授权码，也就是上面经过重定向返回的code。因此我们必须监听这个重定向url。在`WebView`里面，我们可以设置一个`WebViewClient`，重写里面的`onPageFinished()`方法，即可对重定向的地址进行用户点击状态的判断（允许或拒绝）以及状态码的截取。关键代码如下：

```java
webview.setWebViewClient(new WebViewClient() {
    @Override
    public void onPageFinished(WebView view, String url) {
        super.onPageFinished(view, url);
        if (url.contains("?code=")) {
            // Get code and continue to auth
        } else if (url.contains("error=access_denied")) {
            // User disallow
        }
    }
});
```

在取得了授权码code之后，就可以进行后续Oauth2.0中token的请求以及Google API调用了。

参考：
https://developers.google.com/identity/protocols/OAuth2InstalledApp
https://www.learn2crack.com/2014/01/android-oauth2-webview.html

[1]: https://developers.google.com/drive/v2/web/scopes
