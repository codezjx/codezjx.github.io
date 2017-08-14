---
title: Gradle Library Module的复用机制
date: 2015-10-31 11:50:20
categories: Gradle
tags: [Android, Gradle]
---

## 场景
这里有两个AS项目，分别为`projectA`和`projectB`，结构分别如下：
```
projectA/
    ├----build.gradle
    ├----settings.gradle
    ├----bluewhale/
    ├----krill/
```
projectA`settings.gradle`内容：`include 'bluewhale', 'krill'`

```
projectB/
    ├----build.gradle
    ├----settings.gradle
    ├----hello/
    ├----krill/
```
projectB`settings.gradle`内容：`include 'hello', 'krill'`

你可以注意到`projectA`和`projectB`均包含相同的module`krill`，实际上他是一个相同的Library Project，那么问题就来了：如何高效的复用现有的module？实际开发中这个问题应该比较常见，特别针对于同时需要开发多个应用的场景，解决方法如下：

## 单个Library Module复用
- 假设当前Project为GradleProject，在其settings.gralde中include项目的时候指定`projectDir`目录位置，如：
```gradle
include ':LibraryA'
project(':LibraryA').projectDir = new File('../LibraryProject/LibraryA')
include ':LibraryB'
project(':LibraryB').projectDir = new File('../LibraryProject/LibraryB')
```
- 然后在某个App Module中添加dependencies 即可：
```gradle
dependencies {
    ...
    compile project(':LibraryA')
    compile project(':LibraryB')
}
```
- 最后的情况如下截图：
![pic1](20151031114742906.png)

## 多个Library Module统一管理
上面的方式会有一个问题，依赖的library module都是以Project的形式导入的，如果引用越来越多library，外面的Project列表就会越来越长，这个时候我们可以把library module都放在一个Project中统一管理起来。

- 首先新建一个LibraryProject目录，将用到的library module放进去，如LibraryA、LibraryB...

- 拷贝一个默认的build.gradle过去：
```gradle
// Top-level build file where you can add configuration options common to all sub-projects/modules.

buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:1.3.0'

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

allprojects {
    repositories {
        jcenter()
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
```
- settings.gradle内加入：
```gradle
include ':LibraryA', ':LibraryB'，
```
- 最终目录结构如下
```
LibraryProject/
    ├----build.gradle
    ├----settings.gradle
    ├----LibraryA/
    ├----LibraryB/
```

- 最后修改一下GradleProject中的settings.gralde
```gradle
...
// Root project of common library
include ':LibraryProject'
project(':LibraryProject').projectDir = new File('../LibraryProject/')
// Include library module we need
include ':LibraryProject:LibraryA'
include ':LibraryProject:LibraryB'
```
- 最后的最后在GradleProject中的某个App Module中添加dependencies 即可（注意要添加:LibraryProject路径）：
```gradle
dependencies {
    ...
    compile project(':LibraryProject:LibraryA')
    compile project(':LibraryProject:LibraryB')
}
```
- 最后的情况如下截图：
![pic2](20151031114820961.png)

**（注意GradleProject和LibraryProject在同一个目录中哦！）**

## 参考
http://www.philosophicalhacker.com/2014/10/02/an-alternative-multiproject-setup-for-android-studio/
https://docs.gradle.org/current/userguide/build_lifecycle.html#sec:multi_project_builds

下面这个是博主在SO上面的提问&回答，觉得OK请点个up vote哦
http://stackoverflow.com/questions/31910947/how-to-reuse-the-submodule-in-gradle
