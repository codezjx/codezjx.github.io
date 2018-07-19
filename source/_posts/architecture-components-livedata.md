---
title: Architecture Components之LiveData初探
date: 2018-07-14 00:12:23
categories: Architecture Components
tags: [Android, Architecture Components, LiveData]
---

## 简介
`LiveData`是可观察的数据持有者，一般情况下与`ViewModel`搭配使用。它也是典型的观察者模式实现，但是更先进，因为它具有生命周期感知，能确保仅在组件处于活动状态的时候才更新数据，并且在组件销毁的时候能自动解绑，以防止内存泄漏。

## LiveData的优点
- 没有内存泄漏：观察者对象是跟`Lifecycle`对象绑定在一起的，当处于destroyed状态的时候会自动释放资源并解绑。
- 当`Lifecycle`组件处于inactive状态时不会收到事件。
- 不需要手动管理生命周期，`LiveData`将自动进行管理。
- 始终保持数据是最新的，当`Lifecycle`对象从inactive变为active状态的时候，能立刻受到最新的事件。
- 当`Configuration`改变导致界面重建的时候，能立即收到最新的事件。
- 可以使用单例模式扩展`LiveData`进行资源共享。

>当`Lifecycle`组件处于`STARTED`或者`RESUMED`的时候，为active状态。

## 使用LiveData
使用`LiveData`对象包装数据，一般情况下我们会把数据存储在`ViewModel`中使用，然后把包装对象通过`getter`方法返回：
```java
public class NameViewModel extends ViewModel {
    private MutableLiveData<String> mCurrentName;
    public MutableLiveData<String> getCurrentName() {
        if (mCurrentName == null) {
            mCurrentName = new MutableLiveData<String>();
        }
        return mCurrentName;
    }
    ...
}
```

向`LiveData`对象注册观察者，并在回调中更新UI：
```java
// Get the ViewModel.
mViewModel = ViewModelProviders.of(this).get(NameViewModel.class);
mViewModel.getCurrentName().observe(this, new Observer<String>() {
    @Override
    public void onChanged(@Nullable final String newName) {
        // Update the UI
    }
});
```

`MutableLiveData`是`LiveData`的子类，暴露了`setValue(T)`与`postValue(T)`方法来更新数据，前者在主线程调用，后者在子线程调用。 
```java
mViewModel.getCurrentName().setValue(anotherName);
```

## 对`LiveData`进行变换
如果你需要在分发前对数据进行修改或者希望返回一个基于前一个值修改的`LiveData`，你可以使用`Transformations`机制，他类似于RxJava中的操作符变换，允许数据在分发前进行修改。

### Transformations.map()
类似于RxJava中的`map()`操作符，允许对某种类型的数据I转换成数据O。在下面的例子中，`userLiveData`的每次改变，都会触发`userName`的更新。
```java
LiveData<User> userLiveData = ...;
LiveData<String> userName = Transformations.map(userLiveData, new Function<User, String>() {
            @Override
            public String apply(User inputUser) {
                return inputUser.name + " " + inputUser.lastName;
            }
        });
```

### Transformations.switchMap()
与`map()`操作符类似，在下面的例子中，`userIdLiveData`的每次改变，都会导致`getUser(inputId)`被调用，但与`map()`中例子不一样的是这里的`getUser(inputId)`返回的数据是`LiveData`（如：从`Room`数据库中获取数据），因此当每次`getUser(inputId)`返回的`LiveData`数据被修改（如：`Room`中`inputId`对应的`User`被修改），`userLiveData`也会被修改。也就是说`userLiveData`的变化同时依赖`userIdLiveData`与`getUser(inputId)`返回`LiveData`对象的修改。
```java
private LiveData<User> getUser(String id) {
    // Get user from Room database
    ...;
}
LiveData<String> userIdLiveData = ...;
LiveData<User> userLiveData = Transformations.switchMap(userIdLiveData, new Function<String, LiveData<User>>() {
            @Override
            public LiveData<User> apply(String inputId) {
                return getUser(inputId);
            }
        });
```
## 合并多个LiveData源
`MediatorLiveData`是`LiveData`的子类，我们可以通过它对多个`LiveData`源进行合并。在这个过程中，`MediatorLiveData`既是观察者又是被观察者。当其中的一个`LiveData`源变化了，就可以触发`MediatorLiveData`的观察者进行新事件的接收。

一个常见的例子：我们可以同时组合来自数据库的`LiveData`和来自网络的`LiveData`，并通过`MediatorLiveData`添加这两个源。客户端的代码只需观察`MediatorLiveData`对象就好了，不必关心底层数据到底是从哪里来的，参考Repository模式。

这里要注意的一点就是，`MediatorLiveData`在源`LiveData`变化后并不会帮我们实现自动通知，这部分逻辑需要我们手动实现，具体我们可以参考如下官方demo。

以下的案例显示了如何从磁盘显示数据同时从网络加载数据，并及时把网络更新后的数据显示到UI上，以下为整个过程的流程图：

![network-bound-resource](network-bound-resource.png)

整个流程从观察数据库的变化开始，当第一次加载完数据库中的数据时，检查数据是否是有效的，如果有效则直接加载到UI上显示，否则我们将从网络上加载。我们要注意，从数据库加载与网络加载是同时进行的，这里我们先把数据库中缓存的cache显示到UI上，这样用户体验会更好一些。最终我们只需要通过`getAsLiveData()`获取到组合的`MediatorLiveData`即可完成对本地与网络数据变化的监听。

```java
public abstract class NetworkBoundResource<ResultType, RequestType> {
    private final MediatorLiveData<Resource<ResultType>> result = new MediatorLiveData<>();

    @MainThread
    NetworkBoundResource() {
        result.setValue(Resource.loading(null));
        LiveData<ResultType> dbSource = loadFromDb();
        result.addSource(dbSource, data -> {
            result.removeSource(dbSource);
            if (shouldFetch(data)) {
                fetchFromNetwork(dbSource);
            } else {
                result.addSource(dbSource,
                        newData -> result.setValue(Resource.success(newData)));
            }
        });
    }

    private void fetchFromNetwork(final LiveData<ResultType> dbSource) {
        LiveData<ApiResponse<RequestType>> apiResponse = createCall();
        // we re-attach dbSource as a new source,
        // it will dispatch its latest value quickly
        result.addSource(dbSource,
                newData -> result.setValue(Resource.loading(newData)));
        result.addSource(apiResponse, response -> {
            result.removeSource(apiResponse);
            result.removeSource(dbSource);
            //noinspection ConstantConditions
            if (response.isSuccessful()) {
                saveResultAndReInit(response);
            } else {
                onFetchFailed();
                result.addSource(dbSource,
                        newData -> result.setValue(
                                Resource.error(response.errorMessage, newData)));
            }
        });
    }

    @MainThread
    private void saveResultAndReInit(ApiResponse<RequestType> response) {
        ...
    }

    public final LiveData<Resource<ResultType>> getAsLiveData() {
        return result;
    }
}
```