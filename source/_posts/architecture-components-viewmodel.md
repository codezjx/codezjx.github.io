---
title: Architecture Components之ViewModel源码分析
date: 2018-07-13 01:39:00
categories: Architecture Components
tags: [Android, Architecture Components, ViewModel, RTFSC]
---

## 简介
`ViewModel`的作用如它的名字所示，用来存储`View`相关的数据，并且他会自动感知当前界面的生命周期，以便在界面重建的时候能够及时恢复数据（如屏幕旋转、语言等Configuration变化的时候）

## 一分钟上手
我们来简单看下他的使用方法，真的非常简单，首先实现一个自定义的`ViewModel`（先忽略`LiveData`，可把它理解为一个数据的包装器）：
```java
public class CustomViewModel extends ViewModel {
    private MutableLiveData<List<User>> users;
    public LiveData<List<User>> getUsers() {
        if (users == null) {
            users = new MutableLiveData<List<User>>();
            loadUsers();
        }
        return users;
    }
}
```

随后使用`ViewModelProvider.get(Class<T>)`方法即可获取到对应的`ViewModel`实例，结束，就是这么简单。当界面重建并再次调用此方法获取`ViewModel`的时候，将会拿到之前`Activity`创建的实例。
```java
CustomViewModel model = ViewModelProviders.of(this).get(CustomViewModel.class);
```

## RTFSC
是不是觉得很神奇，当界面重建的时候，数据被存到哪了呢？让我们一同翻翻它的源码来找答案~

### ViewModel的生命周期
先来简单熟悉下主角`ViewModel`的生命周期，可以看到，在界面真正`onDestroy()`之前，`ViewModel`是会一直存活的。也就是说`ViewModel`的生命周期会比`Activity`的长，为了避免内存泄漏，我们千万不能随便持有`Activity`的引用。
![viewmodel-lifecycle](viewmodel-lifecycle.png)

### ViewModel vs AndroidViewModel
首先看下`ViewModel`与`AndroidViewModel`这两个类，实现都非常简单。其中`onCleared()`方法会在`ViewModel`不再使用并即将销毁的时候调用。`AndroidViewModel`继承`ViewModel`，比它多了一个`Application`成员变量。因此，当我们需要在`ViewModel`中使用`Context`的时候，就必须选用`AndroidViewModel`，否则就很容易出现内存泄漏。
```java
public abstract class ViewModel {
    protected void onCleared() {
    }
}

public class AndroidViewModel extends ViewModel {
    private Application mApplication;
    public AndroidViewModel(@NonNull Application application) {
        mApplication = application;
    }
    public <T extends Application> T getApplication() {
        //noinspection unchecked
        return (T) mApplication;
    }
}
```

### ViewModelProvider
回到生成`ViewModel`实例的那段代码`ViewModelProviders.of(this).get(CustomViewModel.class)`，我们是通过`ViewModelProviders`的`of(FragmentActivity)`方法来获取`ViewModelProvider`。
```java
public static ViewModelProvider of(@NonNull FragmentActivity activity) {
    return of(activity, null);
}
public static ViewModelProvider of(@NonNull FragmentActivity activity,
        @Nullable Factory factory) {
    Application application = checkApplication(activity);
    if (factory == null) {
        factory = ViewModelProvider.AndroidViewModelFactory.getInstance(application);
    }
    return new ViewModelProvider(ViewModelStores.of(activity), factory);
}
```

当我们没有提供`Factory`的时候，默认会使用`AndroidViewModelFactory`来生成`ViewModel`对象，其中`Factory`是一个工厂方法接口，用于产生对应类型的`ViewModel`对象。
```java
public interface Factory {
    <T extends ViewModel> T create(@NonNull Class<T> modelClass);
}
```

可以看到`AndroidViewModelFactory`的实现十分简单粗暴，如果class是`AndroidViewModel`类型的，就直接调用带`Application`的构造器new对象，否则调用默认构造器new对象。若我们的`ViewModel`需要更多的参数才能创建，在上一步的`of(FragmentActivity)`方法中就必须传入自定义的`Factory`类了。
```java
public static class AndroidViewModelFactory extends NewInstanceFactory {
    private Application mApplication;
    public AndroidViewModelFactory(@NonNull Application application) {
        mApplication = application;
    }
    @NonNull
    @Override
    public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
        if (AndroidViewModel.class.isAssignableFrom(modelClass)) {
            //noinspection TryWithIdenticalCatches
            try {
                return modelClass.getConstructor(Application.class).newInstance(mApplication);
            } catch (XXXException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            }
        }
        return super.create(modelClass);
    }
}

public static class NewInstanceFactory implements Factory {
    @SuppressWarnings("ClassNewInstance")
    @NonNull
    @Override
    public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
        //noinspection TryWithIdenticalCatches
        try {
            return modelClass.newInstance();
        } catch (XXXException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        }
    }
}
```

成功创建了`ViewModelProvider`后，我们就可以通过`get(Class<T>)`方法来获取一个`ViewModel`了。从实现来看也非常简单，通过key从`ViewModelStore`中拿出对象，如果不存在则创建一个新的丢进去并返回。
```java
public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
    String canonicalName = modelClass.getCanonicalName();
    if (canonicalName == null) {
        throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
    }
    return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
}

public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
    ViewModel viewModel = mViewModelStore.get(key);
    if (modelClass.isInstance(viewModel)) {
        //noinspection unchecked
        return (T) viewModel;
    } else {
        //noinspection StatementWithEmptyBody
        if (viewModel != null) {
            // TODO: log a warning.
        }
    }
    viewModel = mFactory.create(modelClass);
    mViewModelStore.put(key, viewModel);
    //noinspection unchecked
    return (T) viewModel;
}
```

### ViewModelStore
接下来看`ViewModelStore`的实现，从类名来看就是一个存储`ViewModel`的地方，实现也还真的非常简单，采用了`HashMap`来存储对象。
```java
public class ViewModelStore {
    private final HashMap<String, ViewModel> mMap = new HashMap<>();
    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }
    final ViewModel get(String key) {
        return mMap.get(key);
    }
    ...
}
```

我们回去看看`ViewModelStore`到底是怎么被创建的，是通过`ViewModelStores.of(FragmentActivity)`方法创建的。我们来瞧一瞧它的实现代码：
```java
public static ViewModelStore of(@NonNull FragmentActivity activity) {
    if (activity instanceof ViewModelStoreOwner) {
        return ((ViewModelStoreOwner) activity).getViewModelStore();
    }
    return holderFragmentFor(activity).getViewModelStore();
}
```

这里需要区分下Support Library的版本，如果在v27.1.0版本及以上的，`FragmentActivity`已经实现了`ViewModelStoreOwner`接口，所以返回的是`activity.getViewModelStore()`
```java
public ViewModelStore getViewModelStore() {
    if (getApplication() == null) {
        throw new IllegalStateException("Your activity is not yet attached to the "
                + "Application instance. You can't request ViewModel before onCreate call.");
    }
    if (mViewModelStore == null) {
        mViewModelStore = new ViewModelStore();
    }
    return mViewModelStore;
}
```

那么界面在重建的时候是如何保存这个`ViewModelStore`的呢？关键就在`onRetainNonConfigurationInstance()`这个方法，系统会自动将其存起来，然后`onCreate()`的时候再将其取出。
```java
public final Object onRetainNonConfigurationInstance() {
    if (mStopped) {
        doReallyStop(true);
    }
    Object custom = onRetainCustomNonConfigurationInstance();
    FragmentManagerNonConfig fragments = mFragments.retainNestedNonConfig();
    if (fragments == null && mViewModelStore == null && custom == null) {
        return null;
    }
    NonConfigurationInstances nci = new NonConfigurationInstances();
    nci.custom = custom;
    nci.viewModelStore = mViewModelStore;
    nci.fragments = fragments;
    return nci;
}

protected void onCreate(@Nullable Bundle savedInstanceState) {
    mFragments.attachHost(null /*parent*/);
    super.onCreate(savedInstanceState);
    NonConfigurationInstances nc =
            (NonConfigurationInstances) getLastNonConfigurationInstance();
    if (nc != null) {
        mViewModelStore = nc.viewModelStore;
    }
    ...
}
```

如果在v27.1.0以下的版本，则返回的是`holderFragmentFor(activity).getViewModelStore()`。首先看下`holderFragmentFor()`逻辑，通过`FragmentManager`来获取`HolderFragment`，如果没有就通过`FragmentManager`添加并存储进`mNotCommittedActivityHolders`中。为什么叫NotCommitted呢？因为在commit之后不会马上添加完成，这是一个异步的过程，所以先把他存在`mNotCommittedActivityHolders`中，以免快速再次调用`holderFragmentFor()`方法的时候不能及时拿到commit的`HolderFragment`。

```java
HolderFragment holderFragmentFor(FragmentActivity activity) {
    FragmentManager fm = activity.getSupportFragmentManager();
    HolderFragment holder = findHolderFragment(fm);
    if (holder != null) {
        return holder;
    }
    holder = mNotCommittedActivityHolders.get(activity);
    if (holder != null) {
        return holder;
    }

    if (!mActivityCallbacksIsAdded) {
        mActivityCallbacksIsAdded = true;
        activity.getApplication().registerActivityLifecycleCallbacks(mActivityCallbacks);
    }
    holder = createHolderFragment(fm);
    mNotCommittedActivityHolders.put(activity, holder);
    return holder;
}

private static HolderFragment createHolderFragment(FragmentManager fragmentManager) {
    HolderFragment holder = new HolderFragment();
    fragmentManager.beginTransaction().add(holder, HOLDER_TAG).commitAllowingStateLoss();
    return holder;
}
```

最后一步就是分析`HolderFragment`，到底他做了什么可以在重建的时候保存数据呢？关键就在于`setRetainInstance(true)`这个方法，它可以在`Activity`重建的时候继续存活，以达到保存数据的目的。
```java
public class HolderFragment extends Fragment implements ViewModelStoreOwner {
    ...
    private ViewModelStore mViewModelStore = new ViewModelStore();
    public HolderFragment() {
        setRetainInstance(true);
    }
    public ViewModelStore getViewModelStore() {
        return mViewModelStore;
    }
    ...
}
```

## 总结
至此，`ViewModel`如何在`Activity`重建的时候成功保存数据就分析完毕了。这里还得注意Support Library的版本问题，如果在v27.1.0及以上版本，是通过`FragmentActivity.onRetainNonConfigurationInstance()`方法实现的，而v27.1.0以下的版本则是通过`Fragment.setRetainInstance(true)`这个方法实现的，大家需要注意区别。还是觉得`onRetainNonConfigurationInstance()`的实现会优雅一点~