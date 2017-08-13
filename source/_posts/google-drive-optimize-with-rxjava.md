---
title: 使用RxJava优化Retrofit请求
date: 2016-04-23 22:28:06
categories: Google Drive Android
tags: [Android, Google Drive, Retrofit, RxJava]
---

在前几篇Google Drive相关的博客中，我们提到了token过期的问题。在进行任何一个Google APIs接口调用的时候，很有可能由于access token过期了（默认的使用期限才3600秒），会导致我们的请求失败，返回HTTP 401: Invalid Credentials error等异常。在这个时候，我们必须重新请求token，然后在请求成功的callback中再次请求我们相关的API。

看到这里，像这种异步的嵌套请求，我们很容易就联想到RxJava，异步世界必不可少的库。那么在Retrofit2.0中如何集成RxJava呢？在基于你已经正常使用Retrofit或者okhttp的情况下，只需简单3步，即可加入RxJava特性。

## 集成RxJava
Retrofit 2.0中加入了CallAdapter机制，官方已经准备好了几个CallAdapter module，其中最著名的module可能是为RxJava准备的`CallAdapter`，它能将请求结果作为`Observable`返回，而不是默认的`Call`对象。

- Step1：对项目加入以下依赖：
```gradle
compile 'com.squareup.retrofit2:adapter-rxjava:2.0.0'
compile 'io.reactivex:rxjava:1.1.3'
compile 'io.reactivex:rxandroid:1.1.0'
```

- Step2：Retrofit Builder链表中加入`RxJavaCallAdapterFactory`
```java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl(BASE_URL)
    .addConverterFactory(GsonConverterFactory.create())
    .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
    .build();
```

- Step3：修改Retrofit的网络请求接口，将返回Call改为返回Observable，现在你可以像操作RxJava中对象一样操作数据流了！
```java
public interface DriveApi {
    @GET("oauth2/v3/userinfo")
    Observable<UserInfo> requestUserInfo(
        @Header("Authorization") String authToken);
}
```

## 使用RxJava优化
根据上面的描述（以请求用户信息这个需求来做例子）：当token失效时，请求会失败。此时我们需要重新请求token，然后再请求用户信息。这里能提取以下两点需求：

- 希望能够在请求用户信息失败时，自动请求token，并及时更新
- 希望能有重试次数限制，防止处于一直请求失败的死循环中

那么这两点在RxJava中能肿么实现呢？

- 在[扔物线][1]的[RxJavaSamples][2]里面认识到了[retryWhen()][3]这个操作符。它可以实现 token 失效时的自动重新获取，将 token 获取的流程彻底透明化，简化开发流程。
- 配合[zipWith()][4]和[range()][5]，可以实现失败重试的次数限制，防止无限重试。

对这3个操作符的原理和用法感兴趣的朋友可以直接进入链接深入学习，下面是经过RxJava优化过的代码（以请求用户信息这个为例子）：
```java
Observable.just(null)
        .flatMap(new Func1<Object, Observable<UserInfo>>() {
            @Override
            public Observable<UserInfo> call(Object o) {
                return mDriveApi.requestUserInfo(mToken.getAccessToken());
            }
        })
        .retryWhen(new Func1<Observable<? extends Throwable>, Observable<?>>() {
            @Override
            public Observable<?> call(Observable<? extends Throwable> observable) {
                // Here we just retry 3 time
                return observable
                        .zipWith(Observable.range(1, 3), new Func2<Throwable, Integer, Throwable>() {
                            @Override
                            public Throwable call(Throwable throwable, Integer integer) {
                                return throwable;
                            }
                        })
                        .flatMap(new Func1<Throwable, Observable<?>>() {
                            @Override
                            public Observable<?> call(Throwable throwable) {
                                if (throwable instanceof HttpException) {
                                    // Request token refresh if request UserInfo fail
                                    return mDriveApi.requestTokenRefresh(mToken.getRefreshToken(),
                                            CLIENT_ID, CLIENT_SECRET, REFRESH_TOKEN)
                                            .doOnNext(new Action1<Token>() {
                                                @Override
                                                public void call(Token token) {
                                                    // Refresh the access token when get success
                                                    mToken.setAccessToken(token.getAccessToken());
                                                }
                                            });
                                }
                                return Observable.just(throwable);
                            }
                        });
            }
        })
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(new Action1<UserInfo>() {
            @Override
            public void call(UserInfo info) {
                // Handle the UserInfo here
            }
        });
```

下面简单解析上述流程：
当`requestUserInfo()`抛出异常的时候，会进入到`retryWhen()`包含的逻辑中。其中throwable Observable通过`zipWith()`与`Observable.range(1, 3)`生成的Observable组合在一起，以实现重试次数最多3次的限制。经过`flatMap()`将`requestTokenRefresh()`返回的Observable return。其中`doOnNext()`操作符实现获取到token后将其更新至相应变量中，以使我们重新`requestUserInfo()`时能用最新的token去请求数据。

有同学对一开始的`Observable.just(null).flatMap()`可能不是太理解，因为token过期经过重试重新请求完回调时，会再次调用`call()`方法重新执行`requestUserInfo()`，并且此时`call()`中传入的object每次都是同一个。若此处去掉`flatMap()`，会导致一直使用旧的token进行请求，所以此处多加了一次`flatMap()`包裹着。还是不理解的，可以把它去掉，自己对比下效果就明白了。

上面的代码用Lambda表达式简化后逻辑会更清晰些，可通过导入[retrolambda][6]插件实现，关于如何导入可以参考[这篇文章][7]。


[1]: http://weibo.com/rengwuxian
[2]: https://github.com/rengwuxian/RxJavaSamples
[3]: http://reactivex.io/documentation/operators/retry.html
[4]: http://reactivex.io/documentation/operators/zip.html
[5]: http://reactivex.io/documentation/operators/range.html
[6]: https://github.com/evant/gradle-retrolambda
[7]: https://codezjx.github.io/2016/05/05/android-retrolambda/
