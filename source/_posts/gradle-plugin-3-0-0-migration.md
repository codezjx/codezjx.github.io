---
title: Android Studio 3.0及Gradle Plugin 3.0升级注意事项
date: 2017-11-23 20:14:20
categories: Gradle
tags: [Android, Android Studio, Gradle, Gradle Plugin]
---

最近终于有空升级了一下项目中的`Gradle`和`Gradle Plugin`的版本，还是踩了蛮多的坑。特别是依赖以及渠道编译这块变动较大，因此把遇到的一些问题点记录下来，分享给后人查阅~

## Gradle版本升级

其实当AS升级到3.0之后，Gradle Plugin和Gradle不升级也是可以继续使用的，但很多新的特性如：Java8支持、新的依赖匹配机制、AAPT2等新功能都无法正常使用~  所以长期看来，最后还是得升的。

- Gradle Plugin升级到`3.0.0`及以上，修改`project/build.gradle`文件：
```gradle
buildscript {
    repositories {
        ...
        // You need to add the following repository to download the
        // new plugin.
        google()
    }

    dependencies {
        classpath 'com.android.tools.build:gradle:3.0.0'
    }
}
```

- Gradle升级到`4.1`及以上，修改`project/gradle/gradle-wrapper.properties`文件：
```
distributionUrl=https\://services.gradle.org/distributions/gradle-4.1-all.zip
```

## 生成APK文件名属性`outputFile`变为只读

改完第一步后会提示如下报错：
>Error:(88, 0) Cannot set the value of read-only property 'outputFile' for ApkVariantOutputImpl_Decorated{apkData=Main{type=MAIN, fullName=appDebug, filters=[]}} of type com.android.build.gradle.internal.api.ApkVariantOutputImpl.

之前改apk名字的代码类似：
```gradle
applicationVariants.all { variant ->
    variant.outputs.each { output ->
        def file = output.outputFile
        def apkName = 'xxx-xxx-xxx-signed.apk'
        output.outputFile = new File(file.parent, apkName)
    }
}
```

由于`outputFile`属性变为只读，需要进行如下修改，直接对`outputFileName`属性赋值即可：
```gradle
applicationVariants.all { variant ->
    variant.outputs.all {
        def apkName = 'xxx-xxx-xxx-signed.apk'
        outputFileName = apkName
    }
}
```

## 依赖关键字的改变

- api: 对应之前的`compile`关键字，功能一模一样。会传递依赖，导致gradle编译的时候遍历整颗依赖树
- implementation: 对应之前的`compile`，与`api`类似，关键区别是不会有依赖传递
- compileOnly: 对应之前的`provided`，依赖仅用于编译期不会打包进最终的apk中
- runtimeOnly: 对应之前的'apk'，与上面的`compileOnly`相反

关于`implementation`与`api`的区别，主要在依赖是否会传递上。如：A依赖B，B依赖C，若使用`api`则A可以引用C，而`implementation`则不能引用。

这里更推荐用`implementation`，一是不会间接的暴露引用，清晰知道目前项目的依赖情况；二是可以提高编译时依赖树的查找速度，进而提升编译速度。详见SO的这个回答，讲得非常详细了：https://stackoverflow.com/questions/44413952/gradle-implementation-vs-api-configuration

## 渠道需要声明flavor dimensions

刚开始Sync的时候应该会报错：
>Error:All flavors must now belong to a named flavor dimension. Learn more at https://d.android.com/r/tools/flavorDimensions-missing-error-message.html

也就是每个flavor渠道都必须归属一个dimension维度，若只有一个维度，渠道中可以不写dimension属性，默认分配到该维度。直接添加一个默认的维度即可，如：`flavorDimensions "dimension"`。当然`flavorDimensions`也可以设置多个维度，详见官方实例：
```gradle
// Specifies two flavor dimensions.
flavorDimensions "mode", "minApi"

productFlavors {
    free {
        // Assigns this product flavor to the "tier" flavor dimension. Specifying
        // this property is optional if you are using only one dimension.
        dimension "mode"
        ...
    }

    paid {
        dimension "mode"
        ...
    }

    minApi23 {
        dimension "minApi"
        ...
    }

    minApi18 {
        dimension "minApi"
        ...
    }
}
```

## 库多variant依赖方式的修改

`Gradle plugin 3.0.0+`之后引入了新的variant自动匹配机制，也就是说app的flavorDebug变体会自动匹配library的flavorDebug变体。

回顾一下旧的方式，如果app在某个variant下需要依赖library相应的类型，需要按照下面的方式声明依赖：
```gradle
dependencies {
    // This is the old method and no longer works for local
    // library modules:
    debugCompile project(path: ':library', configuration: 'debug')
    releaseCompile project(path: ':library', configuration: 'release')
}
```

新的方式，gradle会自动感知并匹配对应的variant（前提是app与library中有对应的variant类型）：
```gradle
dependencies {
    // Instead, simply use the following to take advantage of
    // variant-aware dependency resolution. You can learn more about
    // the 'implementation' configuration in the section about
    // new dependency configurations.
    implementation project(':library')
}
```

## 处理app与lib的依赖匹配问题
上面我们了解到新的variant匹配机制，但若app或library中不存在对应的variant类型呢？匹配将如何进行？下面列出了可能出现的几种情形：

### 情形1：app中有某个build type但library却木有

可以通过`matchingFallbacks`属性来设置回退策略，提供可能的匹配列表，代码如下：
```gradle
// In the app's build.gradle file.
android {
    buildTypes {
        debug {}
        release {}
        staging {
            // Specifies a sorted list of fallback build types that the
            // plugin should try to use when a dependency does not include a
            // "staging" build type. You may specify as many fallbacks as you
            // like, and the plugin selects the first build type that's
            // available in the dependency.
            matchingFallbacks = ['debug', 'qa', 'release']
        }
    }
}
```

若希望可以针对app的每个build type都执行相同的回退策略（例如我们大量的library只有一个release的build type），则可以使用批量指令：
```gradle
buildTypes.all { type ->
    type.matchingFallbacks = ['release']
}
```
**（注意：在该情景下，若library中有某个build type但app却木有，不会对app有任何影响）**

### 情景2：在同一个dimension维度下，如：tier。若app中有某个flavor但library却木有：

同样可以通过`matchingFallbacks`属性来设置回退策略，代码如下：
```gradle
// In the app's build.gradle file.
android {
    defaultConfig{
    // Do not configure matchingFallbacks in the defaultConfig block.
    // Instead, you must specify fallbacks for a given product flavor in the
    // productFlavors block, as shown below.
    }
    flavorDimensions 'tier'
    productFlavors {
        paid {
            dimension 'tier'
            // Because the dependency already includes a "paid" flavor in its
            // "tier" dimension, you don't need to provide a list of fallbacks
            // for the "paid" flavor.
        }
        free {
            dimension 'tier'
            // Specifies a sorted list of fallback flavors that the plugin
            // should try to use when a dependency's matching dimension does
            // not include a "free" flavor. You may specify as many
            // fallbacks as you like, and the plugin selects the first flavor
            // that's available in the dependency's "tier" dimension.
            matchingFallbacks = ['demo', 'trial']
        }
    }
}
```
**（注意：在该情景下，若library中有某个flavor但app却木有，不会对app有任何影响）**

### 情景3：library中有某个dimension维度，但app中却没有:

可以通过`missingDimensionStrategy`属性来设置选择策略，代码如下：
```gradle
// In the app's build.gradle file.
android {
    defaultConfig{
    // Specifies a sorted list of flavors that the plugin should try to use from
    // a given dimension. The following tells the plugin that, when encountering
    // a dependency that includes a "minApi" dimension, it should select the
    // "minApi18" flavor. You can include additional flavor names to provide a
    // sorted list of fallbacks for the dimension.
    missingDimensionStrategy 'minApi', 'minApi18', 'minApi23'
    }
    flavorDimensions 'tier'
    productFlavors {
        free {
            dimension 'tier'
            // You can override the default selection at the product flavor
            // level by configuring another missingDimensionStrategy property
            // for the "minApi" dimension.
            missingDimensionStrategy 'minApi', 'minApi23', 'minApi18'
        }
        paid {}
    }
}
```
说明：其中`missingDimensionStrategy`属性的第一个值为dimension维度，后面的Strings为该维度下的渠道flavors。我们可以看下它的函数原型：
```java
public void missingDimensionStrategy(String dimension, String requestedValue);

public void missingDimensionStrategy(String dimension, String... requestedValues);

public void missingDimensionStrategy(String dimension, List<String> requestedValues);
```
**（注意：在该情景下，若app中有某个dimension维度，但library中却没有，不会对app有任何影响）**

### 情景4：若library没有任何dimension和flavor，则不需app做任何flavor的回退处理~

说了这么多种场景，是不是快被绕晕了？其实诸如dimension的声明以及提供匹配回退策略都是为了实现精确的variant匹配。但是这么多的场景咋看之下还是比较晕，在遇到具体的业务依赖场景后再回来看这一块的内容，你会更加的有收获~

## Java8特性的支持
升级到Gradle Plugin 3.0.0之后，一直被诟病的`Jack`已经被官方弃用了，取而代之的是最新的`desugar`方案。

若项目之前用了类似`retrolambda`或者`Jack`这种旧方案的话，会出现以下提示告诉你移除相关的代码：
>Warning:One of the plugins you are using supports Java 8 language features. To try the support built into the Android plugin, remove the following from your build.gradle: apply plugin: 'me.tatarka.retrolambda' To learn more, go to https://d.android.com/r/tools/java-8-support-message.html

启用最新的`desugar`也非常简单，设置一下`sourceCompatibility`和`targetCompatibility`即可：
```gradle
android {
  ...
  // Configure only for each module that uses Java 8
  // language features (either in its source code or
  // through dependencies).
  compileOptions {
    sourceCompatibility JavaVersion.VERSION_1_8
    targetCompatibility JavaVersion.VERSION_1_8
  }
}
```

目前所支持Java8的特性有：

- Lambda expressions
- Method References
- Type Annotations
- Default and static interface methods
- Repeating annotations

**（注意：stream及function包下的api只能在API level 24+以上才可以使用）**

禁用该特性也是分分钟的事情：
```gradle
android.enableDesugar=false
```

官方文档：
https://developer.android.com/studio/write/java8-support.html

## android-apt相关的异常
最后的最后很多同学会遇到以下关于`android-apt`的报错：

解决方法：

- 移除`android-apt`相关的plugin，如：
```gradle
classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
```

- 依赖中的`apt`改成`annotationProcessor`，如：
```gradle
annotationProcessor 'com.android.databinding:compiler:3.0.0'
```

- 如果有用到类似Realm这种第三方的plugin，确保升级到最新版试试（旧版的Realm用的还是`android-apt`），突然发现升级到最新版后api接口被改了，泪奔中...
```gradle
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath "io.realm:realm-gradle-plugin:4.2.0"
    }
}
```

## 更多

还有更多的迁移变化，由于项目中还没涉及到，就先不写了，大家可以参考官方文档：
https://developer.android.com/studio/build/gradle-plugin-3-0-0-migration.html