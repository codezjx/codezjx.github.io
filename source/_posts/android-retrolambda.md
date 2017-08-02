---
title: 在Android上使用Lambda表达式 - retrolambda插件
date: 2016-05-05 23:10:01
categories: Lambda
tags: [Android, Lambda, retrolambda]
---

**Update:** Android Jack编译工具的加入使得我们可以在旧平台上也使用Lambda表达式了，最重要这是官方支持哦，具体内容看我的这篇：[《在Android上使用官方Lambda支持 - Android N & Jack工具（兼容旧平台）》][5]

## 前言
Java8比较大的一个变化是加入了Lambda表达式，一种紧凑的，传递行为的方式。它可以使你的代码更简洁、逻辑更清晰。特别是用Rxjava的时候，将各种数据变换使用Lambda表达式来简化，可以最大化的减少样板代码，使整个数据流的处理逻辑十分清晰（下面会有个例子）。

## retrolambda
retrolambda又是什么呢？它是 Java 5/6/7 中对Java8 Lambda 表达式的非官方兼容方案。因为目前Android的所有版本（除了N Preview），都还不支持java8。

Github上搜索retrolambda，前两个最多星的就是我们的目标了。分别是[evant/gradle-retrolambda][1]和[orfjackal/retrolambda][2]。他们之间是啥关系呢？简单来说，[gradle-retrolambda][1]只是AS的一个gradle插件，他里面也依赖第二个开源库[orfjackal/retrolambda][2]。所以这里我们直接选第一个进行配置。

## 如何配置
- Step1
下载[jdk8][3]（可以与jdk7并存）
- Step2
修改系统环境变量，设置好`JAVA7_HOME`和`JAVA8_HOME`（为了方便下面插件的配置）
- Step3
修改build.gradle文件，加入以下构建脚本即可（详见：[README][4]）：
```gradle
apply plugin: 'me.tatarka.retrolambda'

buildscript {
  repositories {
     mavenCentral()
  }

  dependencies {
     classpath 'me.tatarka:gradle-retrolambda:3.2.5'
  }
}

retrolambda {
    jdk System.getenv("JAVA8_HOME")
    oldJdk System.getenv("JAVA7_HOME")
    javaVersion JavaVersion.VERSION_1_7
}

android {
    ...
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}
```
（注意：若你使用的gradle plugins版本是2.0.0，你应该使用最新的3.3.0-beta4 gradle-retrolambda插件。）

- Step4
项目默认是依赖v2.1.0版本的[retrolambda][2]库，你可以手动更改此依赖（如修改为最新的2.3.0版本，如果不想修改可以直接跳过这步）：
```gralde
dependencies {
  // Latest version 2.3.0
  retrolambdaConfig 'net.orfjackal.retrolambda:retrolambda:2.3.0'
  // Or a local version
  // retrolambdaConfig files('libs/retrolambda.jar')
}
```

- Step5
执行Gradle Sync Project，稍等AS下载好相关插件及依赖库，就可以开始写Lambda表达式了。

## 效果对比
下面是一个简单的例子，分别是加入Lambda表达式前后的对比，大家随意感受一下：

**没有使用Lambda表达式：**
```java
Observable.from(Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10))
    .filter(new Func1<Integer, Boolean>() {
        @Override
        public Boolean call(Integer integer) {
            return integer % 2 == 0;
        }
    })
    .map(new Func1<Integer, Integer>() {
        @Override
        public Integer call(Integer integer) {
            return integer * integer;
        }
    })
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Action1<Integer>() {
        @Override
        public void call(Integer integer) {
            System.out.println(integer);
        }
    });
```

**加入Lambda表达式后：**
```java
Observable.from(Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10))
    .filter(integer -> integer % 2 == 0)
    .map(integer -> integer * integer)
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(integer -> System.out.println(integer));
```
是不是感觉代码更加清爽、样板代码更少、整个数据的处理逻辑更加清晰了？Lambda表达式的语法也不多，学习成本较少，还没尝试的童鞋可以导入到项目里面耍耍了。

[1]: https://github.com/evant/gradle-retrolambda
[2]: https://github.com/orfjackal/retrolambda
[3]: http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html
[4]: https://github.com/evant/gradle-retrolambda/blob/master/README.md
[5]: https://codezjx.github.io/2016/05/06/android-n-lambda-with-jack/