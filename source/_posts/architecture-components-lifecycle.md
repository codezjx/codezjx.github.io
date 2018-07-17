---
title: Architecture Components之Lifecycles初探
date: 2018-07-12 00:24:00
categories: Architecture Components
tags: [Android, Architecture Components, Lifecycles]
---

## 关于Android Jetpack
`Android Jetpack`是在Google I/O 2018中推出的新一代组件、工具和架构指导，旨在加快您的Android应用开发速度。Jetpack组件将现有的支持库与架构组件联系起来，并将它们分成四个类别：
![jetpack](jetpack.png)

## 关于Architecture Components
`Architecture Components`其实早在Google I/O 2017就推出了，是Jetpack中的架构库，可帮助你设计健壮，可测试和可维护的应用程序。主要用于实现管理UI组件生命周期和处理数据持久性相关的功能。
![architecture-components](architecture-components.svg)

## Lifecycles简介
生命周期感知（Lifecycle-aware）组件作为`Architecture Components`中最重要且最基本的组件，可以在诸如`Activity`和`Fragment`中生命周期变化的时候自动调整它的行为。这些组件能让我们写出更好组织，更轻量且更好维护的代码。

## 导入依赖

导入依赖只需要以下两行就好了，包含了`Lifecycles`、`ViewModel`和`LiveData`：
```gradle
// Lifecycles, ViewModel and LiveData
implementation "android.arch.lifecycle:extensions:1.1.1"
annotationProcessor "android.arch.lifecycle:compiler:1.1.1"
```

可以通过`gradle dependencies`查看依赖树，我们可以看到`extensions`模块里面已经包含了`livedata`和`viewmodel`模块，我们也可以按需单独`implementation`某个模块。
```
+--- android.arch.lifecycle:extensions:1.1.1
|    +--- android.arch.lifecycle:runtime:1.1.1 (*)
|    +--- android.arch.core:common:1.1.1 (*)
|    +--- android.arch.core:runtime:1.1.1 (*)
|    +--- com.android.support:support-fragment:26.1.0 -> 28.0.0-alpha3 (*)
|    +--- android.arch.lifecycle:common:1.1.1 (*)
|    +--- android.arch.lifecycle:livedata:1.1.1
|    |    +--- android.arch.core:runtime:1.1.1 (*)
|    |    +--- android.arch.lifecycle:livedata-core:1.1.1 (*)
|    |    \--- android.arch.core:common:1.1.1 (*)
|    \--- android.arch.lifecycle:viewmodel:1.1.1 (*)
```

在`Support Library 26.1.0`及之后的版本中，默认已经添加了`lifecycle`的支持，诸如`AppCompatActivity`、`Fragment`、`LifecycleService`等都实现了`LifecycleOwner`这个接口。
```
+--- com.android.support:appcompat-v7:28.0.0-alpha3
|    +--- com.android.support:support-compat:28.0.0-alpha3
|    |    +--- android.arch.lifecycle:runtime:1.1.1
|    |    |    +--- android.arch.lifecycle:common:1.1.1
```


## Lifecycle

上面提到了support包里面的`Activity`、`Fragment`都已经实现了`LifecycleOwner`这个接口，我们来看下接口的声明，非常简单，只有一个返回`Lifecycle`的方法：
```
public interface LifecycleOwner {
    /**
     * Returns the Lifecycle of the provider.
     */
    Lifecycle getLifecycle();
}
```

那`Lifecycle`又是什么鬼呢？简单来说，它持有了诸如`Activity`或者`Fragment`组件的生命周期状态信息，并且允许其他对象来观察这些状态，比较典型的观察者模式。

在`Lifecycle`里面主要用到了`Event`和`State`两组枚举常量来追踪关联组件的生命周期状态，可以用下面的时序图来直观的展示`Event`是怎样引起当前`State`的变化。
![lifecycle-states](lifecycle-states.png)

## LifecycleObserver

那么如何监听`Lifecycle`中状态的变化呢？我们可以实现`LifecycleObserver`接口并通过`Lifecycle().addObserver(observer)`来实现监控生命周期状态的变化，当组件发出具体`Event`的时候，`Observer`即可收到相应回调。为了避免在回调中组件的生命周期出现异常，`Lifecycle`还提供了`getCurrentState()`获取当前的状态。
```java
class LocationListener implements LifecycleObserver {
    private boolean enabled = false;
    public LocationListener(Context context, Lifecycle lifecycle, Callback callback) {
       ...
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    void start() {
        if (enabled) {
           // connect
        }
    }

    public void enable() {
        enabled = true;
        if (lifecycle.getCurrentState().isAtLeast(STARTED)) {
            // connect if not connected
        }
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    void stop() {
        // disconnect if connected
    }
}
mLifecycleOwner.getLifecycle().addObserver(new LocationListener());
```

通过上面的方式，我们的`LocationListener`已经能自动感知生命周期的变化并做相应的处理，并且能把相关的逻辑从`Activity`或者`Fragment`中解耦出来，我们的依赖也只有`Lifecycle`这个抽象类。

## 实现自定义的LifecycleOwner
上面我们提到在`Support Library 26.1.0`及之后的版本中`Activity`、`Fragment`都已经实现了`LifecycleOwner`这个接口，如果我们自己的某些类需要实现`Lifecycle`的机制，可以实现`LifecycleOwner`接口并使用`LifecycleRegistry`来完成生命周期事件的转发，如下代码所示：
```java
public class CustomActivity extends Activity implements LifecycleOwner {
    private LifecycleRegistry mLifecycleRegistry;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        mLifecycleRegistry = new LifecycleRegistry(this);
        mLifecycleRegistry.markState(Lifecycle.State.CREATED);
    }

    @Override
    public void onStart() {
        super.onStart();
        mLifecycleRegistry.markState(Lifecycle.State.STARTED);
    }

    @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
}
```

## LifecycleRegistry是什么鬼？
从源码上来看，它是`Lifecycle`的实现类，主要用于处理多个observers的添加/删除，以及新增了`handleLifecycleEvent()`及`markState()`方法以实现生命周期事件发送及状态转换。
```java
public class LifecycleRegistry extends Lifecycle {

    @Override
    public void addObserver(@NonNull LifecycleObserver observer) {
        ...
    }

    @NonNull
    @Override
    public State getCurrentState() {
        return mState;
    }

    public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
        State next = getStateAfter(event);
        moveToState(next);
    }

    public void markState(@NonNull State state) {
        moveToState(state);
    }

    @Override
    public void removeObserver(@NonNull LifecycleObserver observer) {
        ...
    }
}
```
