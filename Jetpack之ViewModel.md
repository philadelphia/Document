ViewModelScope是viewModel的管理者，而ViewModelProvider是ViewModel的间接管理者。
我们一般使用的时候都是ViewModel持有LiveData
### 使用
我们一般获取ViewModel对象都是使用ViewModelProvider的get()方法。
在Activity或者Fragment 里调用

    val  viewProvider:ViewModelProvider  = ViewModelProvider(MainActivity@this)
    val viewModel:MainViewModel =  viewProvider .get(MainViewModel::class.java)

首先得到ViewModelProvider对象
    
```
public ViewModelProvider(@NonNull ViewModelStoreOwner owner) {
    this(owner.getViewModelStore(), owner instanceof HasDefaultViewModelProviderFactory
            ? ((HasDefaultViewModelProviderFactory) owner).getDefaultViewModelProviderFactory()
            : NewInstanceFactory.getInstance());
}
```
get方法
```
public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
    //取出modelClass的全限定名
    String canonicalName = modelClass.getCanonicalName();
    if (canonicalName == null) {
        throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
    }
 //  DEFAULT_KEY ="androidx.lifecycle.ViewModelProvider.DefaultKey"
   //生成新的Key
   return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
}
```
最终调用
```
public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
    ViewModel viewModel = mViewModelStore.get(key);
    if (modelClass.isInstance(viewModel)) {
        if (mFactory instanceof OnRequeryFactory) {
            ((OnRequeryFactory) mFactory).onRequery(viewModel);
        }
        return (T) viewModel;
    } else {
        //noinspection StatementWithEmptyBody
 if (viewModel != null) {
            // TODO: log a warning.
 }
    }
    if (mFactory instanceof KeyedFactory) {
        viewModel = ((KeyedFactory) (mFactory)).create(key, modelClass);
    } else {
        viewModel = (mFactory).create(modelClass);
    }
    mViewModelStore.put(key, viewModel);
    return (T) viewModel;
}
```
可以我们传递的是一个
ViewModelStoreOwner对象
Fragment或者Activity就是这样一个对象，因为他们实现了```
ViewModelStoreOwner接口。
就拿Activity的父类ComponentActivity来举例。
```
public class ComponentActivity extends androidx.core.app.ComponentActivity implements
 LifecycleOwner,
        ViewModelStoreOwner,
        HasDefaultViewModelProviderFactory,
        SavedStateRegistryOwner,
        OnBackPressedDispatcherOwner {
```
它的getViewModelStore方法实现如下：
```
@NonNull
@Override
public ViewModelStore getViewModelStore() {
    if (getApplication() == null) {
        throw new IllegalStateException("Your activity is not yet attached to the "
 + "Application instance. You can't request ViewModel before onCreate call.");
    }
    if (mViewModelStore == null) {
        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            // Restore the ViewModelStore from NonConfigurationInstances
 mViewModelStore = nc.viewModelStore;
        }
        if (mViewModelStore == null) {
            mViewModelStore = new ViewModelStore();
        }
    }
    return mViewModelStore;
}
```

看下ViewModelStore的实现
```
public class ViewModelStore {
    // 一个存储ViewModel的hashMap.
    private final HashMap<String, ViewModel> mMap = new HashMap<>();
    //如果当前已经包含ViewModel了，替换就的ViewModel，并回调其onCleared方法。
    //这个方法在只在ViewModelProvieder的get方法中调用，当ViewModelProvieder从自己的ViewModelScore中无法获取ViewModel对象时，它会new一个新的，然后加入自己的ViewModelScore中。
   
    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }
    
    final ViewModel get(String key) {
        return mMap.get(key);
    }
    
    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }
    
    /**
 *  Clears internal storage and notifies ViewModels that they are no longer used. */ 
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
    }
}
```
### 生命周期
ViewModel会在Activity Destroy是清空自己
在ComponentActivity的构造方法中Activity会监听自己的声明周期的ON_DESTROY时间。而我们的Activity都继承自ComponentActivity。
```
getLifecycle().addObserver(new LifecycleEventObserver() {
    @Override
 public void onStateChanged(@NonNull LifecycleOwner source,
            @NonNull Lifecycle.Event event) {
        if (event == Lifecycle.Event.ON_DESTROY) {
        //如果是配置改变，比如屏幕旋转等导致的销毁，不会回调ViewModel的clear()方法。
            if (!isChangingConfigurations()) {
                getViewModelStore().clear();
            }
        }
    }
});
```
这样，当Activity销毁时，ViewModel就可以在自己的onCleared()方法中做一些清空数据的工作，避免的内存泄露。