# JetPack 之Lifecycle

在JetPack出来之前，我们在做一些比如BroadCastReceiver 之类的注册/解注册之类的活动都需要在Activity的onResume和OnPause之中做处理。如果在Activity销毁时忘记解绑则会导致内存泄露的风险，进而可能引发OOM不过google官方现在给Activity提供了Lifecycle的监听的功能，我们只需要监听Activity对应的状态并给它设置一个LifecycleObserver就好。并处理相应的逻辑就好了。

## Lifecycle

```
public abstract class Lifecycle {
   
    @MainThread
    public abstract void addObserver(@NonNull LifecycleObserver observer);

 
    @MainThread
    public abstract void removeObserver(@NonNull LifecycleObserver observer);


    @MainThread
    @NonNull
    public abstract State getCurrentState();

    @SuppressWarnings("WeakerAccess")
    public enum Event {
      	ON_CREATE,
        ON_START,
        ON_RESUME,
        ON_PAUSE,
        ON_STOP,
        ON_DESTROY,
        ON_ANY
    }

   
    @SuppressWarnings("WeakerAccess")
    public enum State {
        DESTROYED,
        INITIALIZED,
        CREATED,
        STARTED,
        RESUMED;

        public boolean isAtLeast(@NonNull State state) {
            return compareTo(state) >= 0;
        }
    }
}
```

Lifecyle 类提供了添加和删除观察者的方法，此外还有两个内部类，Event和Status分别表示生命周期中的时间和状态。

Lifecycle 只有一个直接实现类LifecycleRegistry

负责Lifecycle 的添加，移出观察者。该类的构造方法如下：

```
public LifecycleRegistry(@NonNull LifecycleOwner provider) {
    mLifecycleOwner = new WeakReference<>(provider);
    mState = INITIALIZED;//将当前状态设置为初始化状态
}
```

## LifecycleOwner

现在Android 提供了一个LifecycleOwner接口,提供了一个Lifecycle 的拥有者的概念

```
/**
 * A class that has an Android lifecycle. These events can be used by custom components to
 * handle lifecycle changes without implementing any code inside the Activity or the Fragment.
 *
 * @see Lifecycle
 */
@SuppressWarnings({"WeakerAccess", "unused"})
public interface LifecycleOwner {
    @NonNull
    Lifecycle getLifecycle();
}

```

该接口有以下实现类：

```
public class SupportActivity extends Activity implements LifecycleOwner, Component {
		......
    private LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

    public Lifecycle getLifecycle() {
        return this.mLifecycleRegistry;
    }
    ......
   }
   
  
  //注意改Fragment是support包下面的，原生的Fragment类没有实现该接口
  public class Fragment implements ComponentCallbacks, OnCreateContextMenuListener, LifecycleOwner, ViewModelStoreOwner {
    LifecycleOwner mViewLifecycleOwner;
    ......
    
      public Lifecycle getLifecycle() {
        return this.mLifecycleRegistry;
    }
    
    ......
    
    }
```

当我们在需要关注Activity/Fragment的声明周期时可以调用getLifecycle，前提是该对象是包含生命周期的。

SupportActivity/Fragment 继承了该接口，所以我们日常使用的Activity/Fragment(support包下面的) 只要继承了该类就自动具有了获取生命周期的能力。

从该类的提示可知，如果我们想得到一个View的生命周期，只需要View实现该接口即可。

```
getLifecycle().addObserver(new DefaultLifecycleObserver() {
            @Override
            public void onCreate(@NonNull LifecycleOwner owner) {

            }

            @Override
            public void onStart(@NonNull LifecycleOwner owner) {

            }

            @Override
            public void onResume(@NonNull LifecycleOwner owner) {

            }

            @Override
            public void onPause(@NonNull LifecycleOwner owner) {

            }

            @Override
            public void onStop(@NonNull LifecycleOwner owner) {

            }

            @Override
            public void onDestroy(@NonNull LifecycleOwner owner) {

            }
        });
```



## LifecycleObserver

LifecycleObserver 只是一个空的接口，有一个子接口`FullLifecycleObserver` 共有6个方法。

```

interface FullLifecycleObserver extends LifecycleObserver {

    void onCreate(LifecycleOwner owner);

    void onStart(LifecycleOwner owner);

    void onResume(LifecycleOwner owner);

    void onPause(LifecycleOwner owner);

    void onStop(LifecycleOwner owner);

    void onDestroy(LifecycleOwner owner);
}
```

`FullLifecycleObserver` 又给子类DefaultLifecycleObserver ，DefaultLifecycleObserver依然是一个接口，我们在getLifecycle.addObserver()方法中只需要new 一个该对象并复写我们关注的声明周期回调方法就行了

这里面最重要的就是LifecycleRegistry类。该类在Lifecycle理念里边扮演了极其重要的概念。

到此貌似没有头绪了，那么Activity作为Lifecycle的拥有者，当其生命周期变化时会通知LifecycleObserver 的。那么看看其中有没有处理lifeEvent 的方法，发现了handleLifecycleEvent方法

```
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    State next = getStateAfter(event);
    moveToState(next);
}
```

那么该方法在哪里被调用的呢？

![image-20200710101720393](/Users/meiliwu/Library/Application Support/typora-user-images/image-20200710101720393.png)

可见有四处调用

## LifecycleDispatcher

```
/**
 * When initialized, it hooks into the Activity callback of the Application and observes
 * Activities. It is responsible to hook in child-fragments to activities and fragments to report
 * their lifecycle events. Another responsibility of this class is to mark as stopped all lifecycle
 * providers related to an activity as soon it is not safe to run a fragment transaction in this
 * activity.
 */
class LifecycleDispatcher {}
```

从该类的注释可以看出：当该类创建时，它hookActivity的回调并观察Activity。它复制Activity的子Fragment 并报告它们的声明周期。该类的另一只职责是：一旦该Activity 不安全地执行Fragment 事务时。它将所有与改Activity相关的Lifecycle provider 的Fragment标记为Stopped 状态



## ProcessLifecycleOwner

该类提供了整个Application进程的声明周期回调



## ServiceLifecycleDispatcher

```
/**
 * Helper class to dispatch lifecycle events for a service. Use it only if it is impossible
 * to use {@link LifecycleService}.
 */
```

该类复制提供Service的声明周期，只有当你使用LifecycleService时，所以以后如果想监听Service的声明周期，就需求我们使用继承LifecycleService的子类了。

## ReportFragment

该类是SupportActivity内部的一个分发初始化lifecycle 事件的内部Fragment，我们使用不到，

我们知道

LifecycleDispatcher 和ProcessLifecycleOwner 分别是监听Activity 和Appliction 的生命周期的，

那么它们是什么时候初始化的呢，既然ProcessLifecycleOwner是监听Application的声明周期的，那么它肯定是在Application对象创建的时候初始化的。

```
@RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
public class ProcessLifecycleOwnerInitializer extends ContentProvider {
    @Override
    public boolean onCreate() {
        LifecycleDispatcher.init(getContext());
        ProcessLifecycleOwner.init(getContext());
        return true;
    }
}
```

系统提供了一个ContentProvider ，它存在于/app/build/intermediates/manifests/full/debug/AndroidManifest.xml，是我们应用在构建完成之后完整生成的AndroidManifest.xml文件。其中，我们可以找到Lifecycle-Aware组件在AndroidManifest的定义。

```java
 <provider
        android:name="android.arch.lifecycle.ProcessLifecycleOwnerInitializer"
        android:authorities="com.boohee.one.lifecycle-trojan"
        android:exported="false"
        android:multiprocess="true" />
```

ProcessLifecycleOwnerInitializer是ContentProvider的子类，利用其onCreate()生命周期方法，处理Lifecycle组件初始化。因此，这是一种隐式初始化的方式。

在ProcessLifecycleOwnerInitializer的onCreate()方法中调用了

```
  LifecycleDispatcher.init(getContext());
  ProcessLifecycleOwner.init(getContext());
```

看下LifecycleDispatcher的init（）方法：

```
private static AtomicBoolean sInitialized = new AtomicBoolean(false);

static void init(Context context) {
	//如果已经初始化过了直接return
    if (sInitialized.getAndSet(true)) {
        return;
    }
    然后对该对象设置ActivityLifecycleCallbacks监听
    ((Application) context.getApplicationContext())
            .registerActivityLifecycleCallbacks(new DispatcherActivityCallback());
}
```

首先调用sInitialized的getAndSet方法，改变量是个AtomicBoolean类型，初始值是false

它这里注册的是个DispatcherActivityCallback对象，该类继承自EmptyActivityLifecycleCallbacks

EmptyActivityLifecycleCallbacks又实现了Application.ActivityLifecycleCallbacks接口。

```
@Override
public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
    if (activity instanceof FragmentActivity) {
        ((FragmentActivity) activity).getSupportFragmentManager()
                .registerFragmentLifecycleCallbacks(mFragmentCallback, true);
    }
    ReportFragment.injectIfNeededIn(activity);
}
```

其目的是为了监听Fragment的生命周期

当前的Activity对象如果继承自FragmentActivity，那么其内部的Fragment 会设置FragmentLifecycleCallbacks回调。而对于那些没有继承自FragmentActivity的Activity，我们给其内部设置一个 ReportFragmet来监听其生命周期。

```
// ProcessLifecycleOwner should always correctly work and some activities may not extend
// FragmentActivity from support lib, so we use framework fragments for activities
ProcessLifecycleOwner一般能正常工作，但是一些Activity可能没有继承FragmentActivity,所以我们使用内部Fragment来监听其生命周期
ReportFragment.injectIfNeededIn(activity);
```

## Lifecycle 的分发

从上面的分析可以，当lifecycleOwner 的生命周期事件发生变化时会回调LifecycleRegistry 的handleLifecycleEvent的。在该方法里处理生命周期回调。

```
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    //根据传入的时间得到一个状态值。
    State next = getStateAfter(event);
    moveToState(next);
}
```



```
static State getStateAfter(Event event) {
    switch (event) {
    	//ON_CREATE，ON_STOP 事件发生时将状态设为CREATED
        case ON_CREATE:
        case ON_STOP:
            return CREATED;
     //ON_START，ON_PAUSE 事件发生时将状态设为STARTED
        case ON_START:
        case ON_PAUSE:
            return STARTED;
        case ON_RESUME:
            return RESUMED;
        case ON_DESTROY:
            return DESTROYED;
        case ON_ANY:
            break;
    }
    throw new IllegalArgumentException("Unexpected event value " + event);
}
```

接下来调用moveToState(next);

```
private void moveToState(State next) {
    if (mState == next) {
        return;
    }
    mState = next;
    if (mHandlingEvent || mAddingObserverCounter != 0) {
        mNewEventOccurred = true;
        // we will figure out what to do on upper level.
        return;
    }
    mHandlingEvent = true;
    sync();
    mHandlingEvent = false;
}
```

```
private void sync() {
		//首先获取LifecycleOwner，如果为null，证明当前Activity 已经被垃圾回收了。直接返回
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        Log.w(LOG_TAG, "LifecycleOwner is garbage collected, you shouldn't try dispatch "
                + "new events from it.");
        return;
    }
    //同步没有完成前，一直循环
    while (!isSynced()) {
        mNewEventOccurred = false;
        // no need to check eldest for nullability, because isSynced does it for us.
        if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
            backwardPass(lifecycleOwner);
        }
        Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
        if (!mNewEventOccurred && newest != null
                && mState.compareTo(newest.getValue().mState) > 0) {
            forwardPass(lifecycleOwner);
        }
    }
    mNewEventOccurred = false;
}
```

至此，我们可以知道，当lifecycle发生变化时，handleLifecycleEvent
 会通过 getStateAfter()方法获取当前应处的状态并修改mState值，紧接着遍历所有 ObserverWithState并调用他们的sync方法来同步且通知LifecycleObserver状态发生变化

## 总结

最后来总结这个监测的流程

```
@RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
public class ProcessLifecycleOwnerInitializer extends ContentProvider {
    @Override
    public boolean onCreate() {
        LifecycleDispatcher.init(getContext());
        ProcessLifecycleOwner.init(getContext());
        return true;
    }
}
```

1：在ProcessLifecycleOwnerInitializer首先调用 LifecycleDispatcher.init(getContext());

该方法给每个Acitivity 内部设置一个Fragment。

2： ProcessLifecycleOwner.init(getContext());

```
  static void init(Context context) {
        sInstance.attach(context);
    }
    
    void attach(Context context) {
        mHandler = new Handler();
        mRegistry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);
        Application app = (Application) context.getApplicationContext();
        app.registerActivityLifecycleCallbacks(new EmptyActivityLifecycleCallbacks() {
            @Override
            public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
                ReportFragment.get(activity).setProcessListener(mInitializationListener);
            }

            @Override
            public void onActivityPaused(Activity activity) {
                activityPaused();
            }

            @Override
            public void onActivityStopped(Activity activity) {
                activityStopped();
            }
        });
    }

```



在init 方法中调用attch方法，给Application对象设置ActivityLifecycleCallbacks

当Activity的oncreate()方法调用时,获取Activity对象的ReportFragment。并设置ProcessListener

```
ReportFragment.get(activity).setProcessListener(mInitializationListener);
```

```
//1：获取Activity内部的ReportFragment
static ReportFragment get(Activity activity) {
    return (ReportFragment) activity.getFragmentManager().findFragmentByTag(
            REPORT_FRAGMENT_TAG);
}
//2：setProcessListener
void setProcessListener(ActivityInitializationListener processListener) {
    mProcessListener = processListener;
}
```

然后在ReportFragment对应的声明周期中回调mProcessListener的相关方法并调用dispatch()方法。

比如onStart()方法

```
  @Override
    public void onStart() {
        super.onStart();
       1： dispatchStart(mProcessListener);
       2： dispatch(Lifecycle.Event.ON_START);
    }
   	//方法1： 
     private void dispatchStart(ActivityInitializationListener listener) {
        if (listener != null) {
            listener.onStart();
        }
    }
    
    //方法2
    private void dispatch(Lifecycle.Event event) {
        Activity activity = getActivity();
        if (activity instanceof LifecycleRegistryOwner) {
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }

        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }
```

在方法2 void dispatch(Lifecycle.Event event)中

先得到当前的Activity对象，并根据器类型调用其liftLycye 的handleLifecycleEvent(event)方法，这样就将其生命周期事件回调给了LifecycleObserver中。