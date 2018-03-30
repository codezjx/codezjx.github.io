---
title: 从ContentProvider报SecurityException分析出Android5.0+的一个隐藏大坑
date: 2018-03-30 21:26:20
categories: Android
tags: [Android, ContentProvider, Bug]
---

## 前言

最近在开发A应用的时候对接了合作方的一个B应用，对方很快就把接口文档发了过来，约定好我们之间通过B应用提供的`XXXContentProvider`来获取相关的数据。一切看起来是如此的普通与简单，但是从刚开始调试的那一刻起，诡异的事情就发送了。九十岁老太为何起死回生？数百头母猪为何半夜惨叫？女生宿舍为何频频失窃？超市方便面为何惨招毒手？在这一切的背后，是人性的扭曲，还是道德的沦丧？事件的最后，让我发现了Android系统的一个大坑！滴滴~ 老司机马上开车，带你一同踏上这段难忘的踩坑经历~

## 先简单回顾下Android的permissions机制

在`AndroidManifest.xml`里面声明`ContentProvider`的时候，我们是可以指定对应的`readPermission`与`writePermission`的，这样就可以限制第三方应用程序，必须声明指定的读写权限，才能进行下一步的访问，提高安全性。
```
<provider
    android:name=".provider.XXXContentProvider"
    android:authorities="com.aaa.bbb.ccc.provider.authorities"
    android:readPermission="com.aaa.bbb.ccc.provider.permission.READ_PERM"
    android:writePermission="com.aaa.bbb.ccc.provider.permission.WRITE_PERM"
    android:exported="true"/>
```

但是首先，我们得先通过`<permission/>`定义好相关应用的权限，且你可以通过`android:protectionLevel`来定义权限的访问等级。常用的有以下几种，更多参数介绍详见官网[permission-element][permission]。
- signature: 调用App必须与声明该`permission`的App使用同一签名
- system: 系统App才能进行访问
- normal: 默认值，系统在安装调用App的时候自动进行授权
```
<permission
    android:name="com.aaa.bbb.ccc.provider.permission.READ_PERM"
    android:protectionLevel="normal" />
<permission
    android:name="com.aaa.bbb.ccc.provider.permission.WRITE_PERM"
    android:protectionLevel="normal" />
```

## What the fuck? SecurityException?

在调用App中，通过`<uses-permission />`声明好调用需要的权限，然后通过`getContentResolver().query()`方法进行数据查询，就这么简单两步。这个时候，程序居然崩溃了，抛出了`SecurityException`。这尼玛我不是按照接口文档声明好权限了么？怎么会报安全问题呢？一定是我打开的方式不对。
```
03-29 12:08:12.839 4255-4271/com.codezjx.provider E/DatabaseUtils: Writing exception to parcel
    java.lang.SecurityException: Permission Denial: reading com.aaa.bbb.ccc.XXXProvider uri content://com.aaa.bbb.ccc.xxx/getxxx/ from pid=22529, uid=10054 requires null, or grantUriPermission()
        at android.content.ContentProvider.enforceReadPermissionInner(ContentProvider.java:539)
        at android.content.ContentProvider$Transport.enforceReadPermission(ContentProvider.java:452)
        at android.content.ContentProvider$Transport.query(ContentProvider.java:205)
        at android.content.ContentProviderNative.onTransact(ContentProviderNative.java:112)
        at android.os.Binder.execTransact(Binder.java:500)
```

上面这段Log是在`ContentProvider`所在的应用发出来的，我们都知道`ContentProvider`中的各种操作其实底层都是通过`Binder`进行进程间通信的。如果Server发生异常，会把exception写进reply parcel中回传到Client，然后Client通过`android.os.Parcel.readException()`读出Server的exception，然后抛出来。没错，就是这么暴力~

这个时候我开始怀疑接口文档的准确性了，马上撸起我的[jadx][jadx]对目标apk进行了反编译，查了下对方的`AndroidManifest.xml`文件。里面声明的`permission`的确没错，而且`ContentProvider`的`authorities`属性也是正确的，`exported`属性也是true。

## SecurityException再次出现

当时一下子没细想，为了快点把数据联调好，我们暂时把`permission`给去掉了。哎呀妈，心想这下子可以安心的联调了。没想到，诡异的事情再次发生了。程序运行，`SecurityException`又再次出现了，还是跟上面的Log一模一样。这尼玛权限不都去掉了吗？为什么还报这个异常呢？
>java.lang.SecurityException: Permission Denial: reading com.aaa.bbb.ccc.XXXProvider uri content://com.aaa.bbb.ccc.xxx/getxxx/ from pid=22529, uid=10054 requires null, or grantUriPermission()

仔细分析了上面这段关键的Log，发现`requires null`这个关键的字眼。一般在`ContentProvider`出现权限问题的时候，会通过requires告诉你到底缺了什么`permission`。然而这里为什么是null呢？想想总感觉不对劲。

## Read the Fucking Source Code

合作方告知，当初一直在4.4的机器上调试的，一直没出现过这个问题。这次在5.1的机器上跑，才发现会奔溃。经过了各种尝试与调试（此处省略一万字），还是没能找到报错的原因，甚至曾一度开始怀疑人生了。这个时候，只能去啃啃源码了，看能不能发现什么端倪。

`ContentProvider`的源码位于`frameworks/base/core/java/android/content/ContentProvider.java`，没有系统源码的也可以直接翻SDK的源码文件。直接查看Log中报错的位置`enforceReadPermissionInner()`方法。

这段方法比较短，还是比较好理解的，其实就是在类似`query()`这些操作前会做一个检查，确认调用方是否具有某些`permission`。如果没授权，就会直接抛出`SecurityException`。
```java
/** {@hide} */
protected void enforceReadPermissionInner(Uri uri, IBinder callerToken)
        throws SecurityException {
    final Context context = getContext();
    final int pid = Binder.getCallingPid();
    final int uid = Binder.getCallingUid();
    String missingPerm = null;

    if (UserHandle.isSameApp(uid, mMyUid)) {
        return;
    }

    if (mExported && checkUser(pid, uid, context)) {
        final String componentPerm = getReadPermission();
        if (componentPerm != null) {
            if (context.checkPermission(componentPerm, pid, uid, callerToken)
                    == PERMISSION_GRANTED) {
                return;
            } else {
                missingPerm = componentPerm;
            }
        }

        // track if unprotected read is allowed; any denied
        // <path-permission> below removes this ability
        boolean allowDefaultRead = (componentPerm == null);

        final PathPermission[] pps = getPathPermissions();
        if (pps != null) {
            final String path = uri.getPath();
            for (PathPermission pp : pps) {
                final String pathPerm = pp.getReadPermission();
                if (pathPerm != null && pp.match(path)) {
                    if (context.checkPermission(pathPerm, pid, uid, callerToken)
                            == PERMISSION_GRANTED) {
                        return;
                    } else {
                        // any denied <path-permission> means we lose
                        // default <provider> access.
                        allowDefaultRead = false;
                        missingPerm = pathPerm;
                    }
                }
            }
        }

        // if we passed <path-permission> checks above, and no default
        // <provider> permission, then allow access.
        if (allowDefaultRead) return;
    }

    // last chance, check against any uri grants
    final int callingUserId = UserHandle.getUserId(uid);
    final Uri userUri = (mSingleUser && !UserHandle.isSameUser(mMyUid, uid))
            ? maybeAddUserId(uri, callingUserId) : uri;
    if (context.checkUriPermission(userUri, pid, uid, Intent.FLAG_GRANT_READ_URI_PERMISSION,
            callerToken) == PERMISSION_GRANTED) {
        return;
    }

    final String failReason = mExported
            ? " requires " + missingPerm + ", or grantUriPermission()"
            : " requires the provider be exported, or grantUriPermission()";
    throw new SecurityException("Permission Denial: reading "
            + ContentProvider.this.getClass().getName() + " uri " + uri + " from pid=" + pid
            + ", uid=" + uid + failReason);
}
```

我们来关注下为什么会是requires null，其实就是因为`missingPerm`没有被赋值。再仔细分析，如果下面这大段代码没有被执行的话，那么`missingPerm`就不会被赋值。
```java
if (mExported && checkUser(pid, uid, context)) {
    ......
}
```

前面已经确认过`mExported`肯定是true的，那么没执行的原因就是`checkUser()`方法返回了false。（之前有提到在Android4.4是不会出现这个`SecurityException`的，为什么呢？因为在Android5.0+后`ContentProvider`才增加了这段多用户检查的代码，泪奔~）

我们来看下`checkUser()`这个方法，种种迹象表明，就是因为它返回了false，导致`missingPerm`没赋值，并最终throw了`SecurityException`。
```java
boolean checkUser(int pid, int uid, Context context) {
    return UserHandle.getUserId(uid) == context.getUserId()
            || mSingleUser
            || context.checkPermission(INTERACT_ACROSS_USERS, pid, uid)
            == PERMISSION_GRANTED;
}
```

通过反射与其他方式，我们可以逐个验证`checkUser()`方法中各个boolean条件的值：
- `(UserHandle.getUserId(uid) == context.getUserId())` -> false
- `mSingleUser` -> false
- `(context.checkPermission(INTERACT_ACROSS_USERS, pid, uid) == PERMISSION_GRANTED)` -> false

前面在踩坑的时候，自己写了一套测试的demo，在正常情况下`UserHandle.getUserId(uid) == context.getUserId()`是会返回true的，其中返回的userId都是0（因为我测试机器就一个用户）

种种迹象表明，合作方提供的问题应用中`context.getUserId()`返回值并不是0。在强烈的好奇心驱使下，我又撸起了[jadx][jadx]对目标apk再次进行了反编译，全局搜索了下`getUserId()`方法，发现还真TM有类似的方法，在`BaseApplication`中，有这么一个`getUserId()`方法，用来返回注册用户的id。

而在`ContentProvider`中，`mContext`也就是`Application`这个`Context`实例，也就是说`getUserId()`方法**被无意识的进行了重写**。因此，解决这个`SecurityException`异常最简单的方法就是把`BaseApplication`中的`getUserId()`方法换个名字就好了。至此，整个踩坑经历终于到了尾声。

## 总结

通过这次踩坑，发现了Android系统中一个隐藏的问题。在自定义的`Application`中，如果你声明了`public int getUserId()`这个方法，并且返回的不是当前用户的`userId`，那么你的`ContentProvider`在Android5.0+的机器都会失效。不信？自己试试~
```java
/** @hide */
@Override
public int getUserId() {
    return mBase.getUserId();
}
```

因为这个是一个`@hide`方法，所以通常这个重写行为都是无意识的，IDE并不会提示你重写了`Application`中的这个方法。但如果你比较幸运，刚好用了带[hidden-api][hidden-api]的Android SDK Jar包，那么IDE会给你一个提示，但除了系统应用开发，一般很少人会导入[hidden-api][hidden-api]吧~
>Missing \`@Override\` annotation on \`getUserId()\` more...

好了，这次的分析先到这里，希望大家以后遇到这个诡异的`SecurityException`异常的时候，不至于再跳进这个隐藏的大坑里~ Over~

[permission]: https://developer.android.com/guide/topics/manifest/permission-element.html
[jadx]: https://github.com/skylot/jadx
[hidden-api]: https://github.com/anggrayudi/android-hidden-api