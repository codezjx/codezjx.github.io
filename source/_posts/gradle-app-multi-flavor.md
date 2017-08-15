---
title: Gradle App项目的多渠道打包实现
date: 2015-10-30 20:25:20
categories: Gradle
tags: [Android, Gradle, Flavor]
---

近期项目需要根据客户的要求定制UI，因此需要用到多渠道打包，跟着官方的[Gradle Plugin User Guide][1]教程学习了下，顺便做下笔记。内容主要分为以下几个模块：

## Create Product flavors
多渠道可以让我们灵活的定制一个应用，如UI、包名、versionName、修改Manifest中的内容等
通过以下DSL即可创建渠道flavor1和flavor2：
```gradle
android {
    ...
    productFlavors {
        flavor1 {}
        flavor2 {}
    }
    ...
}
```

## Build Type + Product Flavor = Build Variant
每一个Variant都是由Build Type和Flavor组合而成，共同生成一个Apk
例如默认的Build Type：debug和release，build完后会生成以下4个Variants
Flavor1 - debug
Flavor1 - release
Flavor2 - debug
Flavor2 - release

## Product Flavor Configuration
我们可以在Flavor对应的闭包内自定义每个渠道相关属性
```gradle
android {
    ...
    defaultConfig {
        minSdkVersion 8
        versionCode 10
    }

    productFlavors {
        flavor1 {
            packageName "com.example.flavor1"
            versionCode 20
        }

        flavor2 {
            packageName "com.example.flavor2"
            minSdkVersion 14
        }
    }
}
```
- flavor对象与defaultConfig的类型是相同的，所以他们的属性也一样
- defaultConfig可以作为每个渠道基础的公共配置，渠道中的属性会覆盖defaultConfig

## Sourcesets and Dependencies
Android Studio默认目录结构下，每个渠道会创建自己的sourceSets，与默认的android.sourceSets.main共同编译生成Apk：
```gradle
android.sourceSets.main
- src/main/
android.sourceSets.flavor1
- src/flavor1/
android.sourceSets.flavor2
- src/flavor2/
```
若是Eclipse的那种目录结构，可选择自己指定sourceSets位置，如：
```gradle
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
    flavor1 {
        java.srcDirs = ['xxx-folder/flavor1/src']
        res.srcDirs = ['xxx-folder/flavor1/res']
    }
    flavor2 {
        java.srcDirs = ['xxx-folder/flavor2/src']
        res.srcDirs = ['xxx-folder/flavor2/res']
    }
}
```
编译某个渠道包的时候遵循以下4条准则：

- 所有的源码(src/*/java)会用来共同编译生成一个Apk，不允许覆盖，会提示`duplicate class found`
- 所有的Manifests都将会合并，这样一来就允许渠道包中可以定义不同的组件与权限，具体可参考官方[`Manifest Merger`][2]
- 渠道中的资源会以覆盖或增量的形式与main合并，优先级为`Build Type > Product Flavor > Main sourceSet`
- 每个Build Variant都会生成自己的R文件

渠道可以指定自己特定的dependencies，命名规则为(Flavor Name)Compile
```gradle
dependencies {
    flavor1Compile "..."
}
```

## Building and Tasks
每个Build Variants都会创建相应的assemble task，可以通过以下命名方式调用对应指令来执行编译：

- assemble(Variant Name)   // 编译指定的Variant，如：assembleFlavor1Debug
- assemble(Build Type Name)   // 如：assembleDebug将同时编译Flavor1Debug和Flavor2Debug variants
- assemble(Product Flavor Name)  // 如：assembleFlavor1将同时编译Flavor1Debug和Flavor1Release variants

执行`gradle assemble`将会编译所有的variants包，用得比较多的还是编译所有渠道的release包：`gradle assembleRelease`，然后就去喝杯咖啡慢慢的等待编译完成吧


[1]:http://tools.android.com/tech-docs/new-build-system/user-guide#TOC-Build-Variants
[2]:http://tools.android.com/tech-docs/new-build-system/user-guide/manifest-merger