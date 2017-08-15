---
title: Gradle Library项目的多渠道打包实现
date: 2015-10-30 20:36:20
categories: Gradle
tags: [Android, Gradle, Flavor]
---

项目中由于某种需求需要对Library项目也进行多渠道发布，如：App已经实现了多渠道打包，此时不同渠道包依赖的同一个Library中的某些资源（举个栗子）也需要根据渠道不同而改变，这个时候就需要对Library进行多渠道发布了。

实现起来也比较简单，步骤如下：

## 先对Library进行多渠道发布
```
apply plugin: 'com.android.library'

android {
    ...
    publishNonDefault true
    productFlavors {
        flavor1 {}
        flavor2 {}
    }
    ...
    sourceSets {
        flavor1 {
            res.srcDirs = ['xxx-folder/flavor1/res']
        }
        flavor2 {
            res.srcDirs = ['xxx-folder/flavor2/res']
        }
    }
}
```
注意其中的`publishNonDefault true`，这个是关键，主要是用来设置Library发布所有的variants

为何需要这么设置？请允许我啰嗦几句：默认情况下Library项目只会发布它的release aar包，这也就是为什么库项目中的`BuildConfig.DEBUG`一直是false的原因

可以通过`defaultPublishConfig "debug"`来修改这种默认的发布机制（针对没有渠道的情况）
若已经有多个渠道，则必须指定完整的variant名字，如：`defaultPublishConfig "flavor1Debug"`


## 修改App项目中的dependencies方式
根据App不同的渠道编译Library不同的渠道：
```
dependencies {
    ...
    flavor1Compile project(path: ':lib', configuration: 'flavor1Release')
    flavor2Compile project(path: ':lib', configuration: 'flavor2Release')
    ...
}
```
注意：若遇到异常：Gradle DSL method not found: 'flavor1Compile()'，请把dependencies{}移至build.gradle脚本最下方

## 参考
http://tools.android.com/tech-docs/new-build-system/user-guide#TOC-Library-Publication
http://stackoverflow.com/questions/24860659/multi-flavor-app-based-on-multi-flavor-library-in-android-gradle/24910671#24910671