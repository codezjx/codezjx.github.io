---
title: Google Drive Oauth2.0认证流程
date: 2016-03-31 19:35:06
categories: Google Drive Android
tags: [Android, Google Drive, OAuth]
---

## Oauth2.0认证流程
Google提供的APIs访问是基于Oauth2.0认证的，其流程可以大致分为以下几个步骤：

1. 客户端App发起认证（若用户木有登录，则需要先登录）
2. 弹出授权页面，用户点击允许后，Google认证服务器给我们返回一个授权码（Authorization code）
3. 客户端获取到授权码以后，用授权码向认证服务器请求access token
4. 服务器验证授权码无误后返回access token至客户端
5. 拿到access token以后，就可以访问Google APIs了（其实这里还会返回一个refresh token）

大家直接看下面的图可能会比较好理解一点：

![这里写图片描述](20160331193430106.png)

上面的流程只是基于第一次获取access token的情况，因为access token是有期限的，默认是1个小时，access token过期之后，就需要通过refresh token来向Google认证服务器申请一个新的access token，不需要经历上面的1,2,3步。


## Refresh Token期限
Refresh Token并不是一直有效的，在下面的几种情况下将会失效： 

- 用户回收了授权
- token超过6个月木有被使用
- 用户修改了密码并且token授权scope包含了：Gmail, Calendar, Contacts, 或者Hangouts
- 超过了一定数量的token数量限制

目前每个用户账号每个授权凭证有25个refresh token的数量限制，如果超过了这个限制，当你新建一个refresh  token的时候将会使最早创建那个失效。一般来说，我们在经过用户授权，拿到授权码请求到refresh token后，必须把它缓存起来，以便后续更新access token。


## 参考：

https://developers.google.com/identity/protocols/OAuth2
