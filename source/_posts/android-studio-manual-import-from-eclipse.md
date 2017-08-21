---
title: Android Studio中手动导入Eclipse Project
date: 2014-08-29 21:34:22
categories: Android Studio
tags: [Android, Android Studio]
---

## 导入步骤

RT，这应该是很多朋友刚从Eclipse转到Android Studio后遇到最大的一个问题，首先我们需要重新认识AS里面的目录结构，在我前一篇帖子里面也有提到（**Android Studio中的Project相当于Eclipse中的Workspace，Module则相当于Eclipse中的Project**）。所以我们手动导入Project，其实就是导入AS里面的Module。主要有以下几个步骤：

1. 复制`build.gradle`到需要导入的项目中
2. 复制你需要导入的项目至AS Project根目录文件夹下（即存在gradlew, gradlew.bat, .gradle的那个文件夹）
3. 修改AS Project中的`settings.gradle`，添加include，告诉AS我们的Project里面需要包含这个Module（例如`include ':SlidingMenuLibrary'`）
4. Rebuild Project，会为项目自动生成`.iml`文件（iml文件是AS识别项目的配置文件，跟Eclipse里面的.project文件作用类似）

下面贴出`build.gradle`主要内容：
```gradle
apply plugin: 'com.android.application'

dependencies {
    compile fileTree(dir: 'libs', include: '*.jar')
}

android {
    compileSdkVersion 17
    buildToolsVersion "20.0.0"

    sourceSets {
        main {
            manifest.srcFile 'AndroidManifest.xml'
            java.srcDirs = ['src']
            resources.srcDirs = ['src']
            aidl.srcDirs = ['src']
            renderscript.srcDirs = ['src']
            res.srcDirs = ['res']
            assets.srcDirs = ['assets']
        }

        // Move the tests to tests/java, tests/res, etc...
        instrumentTest.setRoot('tests')

        // Move the build types to build-types/<type>
        // For instance, build-types/debug/java, build-types/debug/AndroidManifest.xml, ...
        // This moves them out of them default location under src/<type>/... which would
        // conflict with src/ being used by the main source set.
        // Adding new build types or product flavors should be accompanied
        // by a similar customization.
        debug.setRoot('build-types/debug')
        release.setRoot('build-types/release')
    }
}
```

## 注意事项

其实上面这个文件就是用Eclipse导出的，大家可以放心的直接复制使用。还有需要注意以下几点：
1. 上面提供的是App的gradle文件，如果是Library项目，则需要修改`apply plugin: 'com.android.application'` 为 `apply plugin: 'com.android.library'`即可
2. `compileSdkVersion` 和 `buildToolsVersion`，需要根据本地的SDK版本具体修改，打开SDK Manager看下就行了
3. `sourceSets main`里面指定了源代码的目录位置，因为AS默认的代码结构与Eclipse的是不一样的
4. `dependencies`里面指定了依赖库，`compile fileTree(dir: 'libs', include: ['*.jar'])`编译libs目录下所有的.jar库。如果依赖某些库项目，则可以添加：`compile project(':Cordova')`
5. 若项目是Git项目，需要AS识别的话，修改`.idea`目录下的`vcs.xml`文件，增加一条`<mapping directory="$PROJECT_DIR$/XXXXProject" vcs="Git" />`即可。然后你对着Module右键，是不是已经发现有Git这个选项了？ 嘿嘿

## 结束语

其实，最新版本的AS已经支持直接导入Module了，但有一个问题，它在导入的时候，会Copy一份你的项目（相当于重新生成一份），然后导入后目录结构就变成了AS的目录结构，如果想保持Eclipse目录结构，还是使用上面的方法吧，嘿嘿。