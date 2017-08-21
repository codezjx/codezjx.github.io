---
title: Android Studio使用心得 - 常见问题集锦
date: 2014-08-19 00:16:16
categories: Android Studio
tags: [Android, Android Studio]
---

整理了一些这段时间遇到的常见问题，希望对各位猿们有帮助。。。如果觉得有用就点个赞哦

## 问题一

```
Error:(26, 9) Attribute application@icon value=(@drawable/logo) from AndroidManifest.xml:26:9
Error:(28, 9) Attribute application@theme value=(@style/ThemeActionBar) from AndroidManifest.xml:28:9
is also present at XXXX-trunk:XXXXLib:unspecified:15:9 value=(@style/AppTheme)
Suggestion: add 'tools:replace="android:theme"' to <application> element at AndroidManifest.xml:24:5 to override
Error:Execution failed for task ':XXXX:processDebugManifest'.
> Manifest merger failed with multiple errors, see logs
```

**原因：**
AS的Gradle插件默认会启用Manifest Merger Tool，若Library项目中也定义了与主项目相同的属性（例如默认生成的`android:icon`和`android:theme`），则此时会合并失败，并报上面的错误。

**解决方法有以下2种：**
- 方法1：在Manifest.xml的application标签下添加`tools:replace="android:icon, android:theme"`（多个属性用,隔开，并且记住在manifest根标签上加入`xmlns:tools="http://schemas.android.com/tools"`，否则会找不到namespace哦）
- 方法2：在build.gradle根标签上加上`useOldManifestMerger true` （懒人方法）

**参考官方介绍：**
http://tools.android.com/tech-docs/new-build-system/user-guide/manifest-merger


## 问题二

Library Project里面的`BuildConfig.DEBUG`永远都是false。这是Android Studio的一个已知问题，某Google的攻城狮说，Library projects目前只会生成release的包。
Issue 52962: https://code.google.com/p/android/issues/detail?id=52962

**解决方法：**（某Google的攻城狮推荐的方法）
Workaround: instaed of BuildConfig.DEBUG create another boolean variable at lib-project's e.g. BuildConfig.RELEASE and link it with application's buildType. 
https://gist.github.com/almozavr/d59e770d2a6386061fcb

**参考stackoverflow上的这篇帖：**
http://stackoverflow.com/questions/20176284/buildconfig-debug-always-false-when-building-library-projects-with-gradle


## 问题三

每次保存的时候，每行多余的空格和TAB会被自动删除（例如结尾、空行的多余空格或TAB）
特别是每次准备提交SVN，Review代码时候你就蛋疼了，显示一堆不相关的更改，看的眼花。

**解决方法：**
Settings->IDE Settings->Editor->Other->Strip trailing spaces on Save->None


## 问题四

编译的时候，报：Failure [INSTALL_FAILED_OLDER_SDK]。一般是系统自动帮你设置了compileSdkVersion 

**解决方法：**
修改`build.gradle`下的`compileSdkVersion 'android-L'`为`compileSdkVersion 19`（或者你本机已有的SDK即可）




