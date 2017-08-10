---
title: 使用Retrofit2.0实现GoogleDrive相关API
date: 2016-04-23 18:32:06
categories: Google Drive Android
tags: [Android, Google Drive, Retrofit]
---

做移动开发的相信对Retrofit一点也不陌生，它是一套RESTful架构的Android(Java)客户端实现，可以利用接口，方法和注解参数（parameter annotations）来声明式定义一个请求应该如何被创建。它的出现使我们只需关注接口所定义的功能而非拘泥于具体实现中，极大简化和提升了开发效率。

## 相关API接口及请求参数
下面我们来看下最近在项目用到的几个API接口和请求参数，并用Retrofit2.0来实现他们：

1.请求token（其中grant_type字段固定值为authorization_code，表示是用authCode来请求token。authCode就是之前我们通过用户授权获取到的授权码，为以下接口中的code字段。）
>POST /oauth2/v4/token HTTP/1.1
Host: www.googleapis.com
Content-Type: application/x-www-form-urlencoded

>code=4/v6xr77ewYqhvHSyW6UJ1w7jKwAzu&
client_id=8819981768.apps.googleusercontent.com&
client_secret=your_client_secret&
redirect_uri=https://oauth2-login-demo.appspot.com/code&
grant_type=authorization_code

2.更新token （其中grant_type字段固定值为refresh_token，表示用refresh token请求一个新的token。为啥要更新token？因为access token是有一定期限的，过期了以后需要用refresh token去请求一个新的token，具体可以看我上一篇文章）

>POST /oauth2/v4/token HTTP/1.1
Host: www.googleapis.com
Content-Type: application/x-www-form-urlencoded

>client_id=8819981768.apps.googleusercontent.com&
client_secret=your_client_secret&
refresh_token=1/6BMfW9j53gdGImsiyUH5kU5RsR4zwI9lUVX-tqf8JXQ&
grant_type=refresh_token

3.获取用户基本信息（直接一个get请求，但注意这里需要在Header中添加Authorization认证信息，也就是我们上面接口返回的access token，目前Google Oauth2.0的token类型都固定为Bearer。token认证的魅力也在于此，无需持有客户的账号与密码即可完成相应请求，当然前提还是需要客户允许才能获取授权码。）
>GET /oauth2/v3/userinfo HTTP/1.1
Host: www.googleapis.com
Authorization: Bearer your_auth_token 
Content-Type: application/x-www-form-urlencoded

4.Drive文件上传（同样需要在Header中添加Authorization认证信息，注意这里涉及到了Multipart的文件上传，其实也有个uploadType=media的简单上传，但是比较坑爹的是采用这种方式上传，在你的GoogleDrive云盘中会显示一个Untitled.jpg的文件。采用Multipart的话，可以分part上传：Metadata part和Media part。Meta part可以指定文件的一些属性如：文件名、保存位置等，Media part主要就是指定上传文件的具体数据了。啰嗦几句：boundary指定了每一part的分隔边界，服务器会根据这个字段来解析我们的每一part。具体可以参考[这篇文章][1]）
>POST /upload/drive/v3/files?uploadType=multipart HTTP/1.1
Host: www.googleapis.com
Authorization: Bearer your_auth_token
Content-Type: multipart/related; boundary=foo_bar_baz
Content-Length: number_of_bytes_in_entire_request_body

>--foo_bar_baz
Content-Type: application/json; charset=UTF-8
{
  "name": "My File"
}
--foo_bar_baz
Content-Type: image/jpeg
JPEG data
--foo_bar_baz--

## Retrofit2.0实现接口定义
根据上面的接口描述我们可以用Retrofit如下定义出接口：（注意`@FormUrlEncoded`和`@Multipart`必不可少否则会抛出异常，`@Part`可以描述Multipart中的某part数据，`@Field`描述Post的某字段）
```java
public interface DriveApi {

    @POST("oauth2/v4/token")
    @FormUrlEncoded
    Call<Token> requestToken(
            @Field("client_id") String clientId,
            @Field("client_secret") String clientSecret,
            @Field("code") String code,
            @Field("redirect_uri") String redirectUri,
            @Field("grant_type") String grantType);

    @POST("oauth2/v4/token")
    @FormUrlEncoded
    Call<Token> requestTokenRefresh(
            @Field("refresh_token") String refreshToken,
            @Field("client_id") String clientId,
            @Field("client_secret") String clientSecret,
            @Field("grant_type") String grantType);

    @GET("oauth2/v3/userinfo")
    Call<UserInfo> requestUserInfo(
            @Header("Authorization") String authToken);

    @POST("upload/drive/v3/files?uploadType=multipart")
    @Multipart
    Call<UploadResult> uploadFileMutil(
            @Header("Authorization") String authToken,
            @Part MultipartBody.Part metaPart,
            @Part MultipartBody.Part dataPart);
}

```

然后通过`Retrofit`对象的`create()`方法即可实例化上述接口，完全不需要关注具体的实现，只需关注接口的参数和返回值就OK了，是不是爽到爆？这里用到了`GsonConverterFactory`，自动将返回的json数据转换为POJO，默认是根据POJO类的字段名实现映射。
```java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://www.googleapis.com/")
    .addConverterFactory(GsonConverterFactory.create())
    .build();
DriveApi driveApi = retrofit.create(DriveApi.class);
```

接下来就是使用DriveApi进行接口的调用了。在Retrofit2.0中，接口的定义统一返回`Call`对象，与1.x版本相比变化还是挺大的。2.0中如果要调用同步请求，只需调用`Call`对象的`execute()`方法，调用`enqueue()`则可以发起一个异步请求。

- 同步请求：
```java
// Synchronous Call in Retrofit 2.0
  
Call<Token> call = driveApi.requestToken();
Token token = call.execute();
```

- 异步请求：
```java
// Asynchronous Call in Retrofit 2.0
  
Call<Token> call = driveApi.requestToken();
call.enqueue(new Callback<Token>() {
    @Override
    public void onResponse(Response<Token> response) {
        // Get result Token from response.body()
    }
  
    @Override
    public void onFailure(Throwable t) {
  
    }
});
```

- 取消请求（直接`cacel()`即可，2.0统一改成返回call对象可以方便的让我们直接取消正在进行的请求）
```java
call.cancel();
```

有关Retrofit2.0的一些改进，建议可以看下面这篇：Retrofit 2.0：[有史以来最大的改进][2]，总结得挺全面的。

最后别忘记添加相关依赖：
```
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile 'com.squareup.okhttp3:okhttp:3.2.0'
    compile 'com.squareup.retrofit2:retrofit:2.0.0'
    compile 'com.squareup.retrofit2:converter-gson:2.0.0'
}
```

[1]:http://blog.csdn.net/xiaojianpitt/article/details/6856536
[2]:http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0915/3460.html
