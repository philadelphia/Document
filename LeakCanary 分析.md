LeakCanary 分析

LeakCanary是Square公司推出个一个内存泄露检测工具，

[地址：[https://square.github.io/leakcanary/]()](https://square.github.io/leakcanary/)



## 使用

简单使用，

在项目Model的build.gradle文件中dependencies{}中加入

```
debugImplementation 'com.squareup.leakcanary:leakcanary-android:version'
```

然后在你的App中的onCreate()方法中添加

```
LeakCanary.install(this);
```

这样LeakCanary就可以监控系统的Activity的内存泄露了，默认使用只能检测到Activity的内存泄露。

如果你想检测其他对象的内存泄露，可以自定义监控对象。

```
refWatcher = LeakCanary.install(this);
```

LeakCanary.install(this) 方法返回一个RefWatcher对象，调用该队的的watch()方法就行了。

该方法的签名如下

```
//传入你想检测的对象
public void watch(Object watchedReference) {
  watch(watchedReference, "");
}

```

#原理

LeakCanary 中对内存泄漏检测的核心原理就是基于 WeakReference 和 ReferenceQueue 实现的。

1. 当一个 Activity 需要被回收时，就将其包装到一个 WeakReference 中，并且在 WeakReference 的构造器中传入自定义的 ReferenceQueue。

2. 然后给包装后的 WeakReference 做一个标记 Key，并且在一个强引用 Set 中添加相应的 Key 记录

3. 最后主动触发 GC，遍历自定义 ReferenceQueue 中所有的记录，并根据获取的 Reference 对象将 Set 中的记录也删除


经过上面 3 步之后，还保留在 Set 中的就是：应当被 GC 回收，但是实际还保留在内存中的对象，也就是发生泄漏了的对象

```
public static RefWatcher install(Application application) {
  return install(application, DisplayLeakService.class);
}

 public static RefWatcher install(Application application,
      Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    if (isInAnalyzerProcess(application)) {
      return RefWatcher.DISABLED;
    }
    enableDisplayLeakActivity(application);
    HeapDump.Listener heapDumpListener =
        new ServiceHeapDumpListener(application, listenerServiceClass);
    RefWatcher refWatcher = androidWatcher(application, heapDumpListener);
  //  LeakCanary 中监听 Activity 生命周期是由 ActivityRefWatch 来负责的，主要是通过注册 Android 系统提供的 ActivityLifecycleCallbacks，来监听 Activity 的生命周期方法的调用，
    ActivityRefWatcher.installOnIcsPlus(application, refWatcher);//默认只检测Activity的内存泄露
    return refWatcher;//返回refWatcher对象，
  }
```

RefWatcher对象的构造方法如下。	

	  private final Executor watchExecutor; //线程池
	  private final DebuggerControl debuggerControl;
	  private final GcTrigger gcTrigger;//GC触发器
	  private final HeapDumper heapDumper;
	  private final Set<String> retainedKeys; //检测对象的Key集合
	  private final ReferenceQueue<Object> queue;//若引用队列
	  private final HeapDump.Listener heapdumpListener;
	  
	public RefWatcher(Executor watchExecutor, DebuggerControl debuggerControl, GcTrigger gcTrigger,HeapDumper heapDumper, HeapDump.Listener heapdumpListener) {
		this.watchExecutor = checkNotNull(watchExecutor, "watchExecutor");
	  this.debuggerControl = checkNotNull(debuggerControl, "debuggerControl");
	  this.gcTrigger = checkNotNull(gcTrigger, "gcTrigger");
	  this.heapDumper = checkNotNull(heapDumper, "heapDumper");
	  this.heapdumpListener = checkNotNull(heapdumpListener, "heapdumpListener");
	  retainedKeys = new CopyOnWriteArraySet<>();
	  queue = new ReferenceQueue<>();
	}


我们看下ActivityRefWatcher这个类的内部实现，该类只支持ICE_CREAM_SANDWICH以上的版本。

```
@TargetApi(ICE_CREAM_SANDWICH) public final class ActivityRefWatcher {


  private final Application application;
  private final RefWatcher refWatcher;
  
  public static void installOnIcsPlus(Application application, RefWatcher refWatcher) {
   低于ICE_CREAM_SANDWICH版本的自己在BaseActivity的onDestroy方法中处理
   if (SDK_INT < ICE_CREAM_SANDWICH) {
      // If you need to support Android < ICS, override onDestroy() in your base activity.
      return;
    }
    ActivityRefWatcher activityRefWatcher = new ActivityRefWatcher(application, refWatcher);
    activityRefWatcher.watchActivities();
  }

  /**
   * Constructs an {@link ActivityRefWatcher} that will make sure the activities are not leaking
   * after they have been destroyed.
   */
  public ActivityRefWatcher(Application application, final RefWatcher refWatcher) {
    this.application = checkNotNull(application, "application");
    this.refWatcher = checkNotNull(refWatcher, "refWatcher");
  }


  public void watchActivities() {
    // Make sure you don't get installed twice.
    stopWatchingActivities();
//   application注册ActivityLifecycleCallbacks接口
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks);
  }

  public void stopWatchingActivities() {
    application.unregisterActivityLifecycleCallbacks(lifecycleCallbacks);
  }
  
  private final Application.ActivityLifecycleCallbacks接口 lifecycleCallbacks =
      new Application.ActivityLifecycleCallbacks() {
        @Override public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
        }

        @Override public void onActivityStarted(Activity activity) {
        }

        @Override public void onActivityResumed(Activity activity) {
        }

        @Override public void onActivityPaused(Activity activity) {
        }

        @Override public void onActivityStopped(Activity activity) {
        }

        @Override public void onActivitySaveInstanceState(Activity activity, Bundle outState) {
        }

				//在Activity销毁时开启检测
        @Override public void onActivityDestroyed(Activity activity) {
          ActivityRefWatcher.this.onActivityDestroyed(activity);
        }
      };
      
          //检测
  void onActivityDestroyed(Activity activity) {
    refWatcher.watch(activity);
  }
      
}
```

最后调用refWatcher对象的watch方法。最后回调它的重载方法。

```
public void watch(Object watchedReference) {
  watch(watchedReference, "");
}

 public void watch(Object watchedReference, String referenceName) {
 	//对检测对象和引用名称判空
    checkNotNull(watchedReference, "watchedReference");
    checkNotNull(referenceName, "referenceName");
    if (debuggerControl.isDebuggerAttached()) {
      return;
    }
    //检测开始时间
    final long watchStartNanoTime = System.nanoTime();
    //随机生成Key
    String key = UUID.randomUUID().toString();
    //将Key放入retainedKeys集合中，retainedKeys是一个CopyOnWriteArraySet对象。
    retainedKeys.add(key);
    //将检测对象，key,referenceName，queue 构建一个KeyedWeakReference对象
    final KeyedWeakReference reference =
        new KeyedWeakReference(watchedReference, key, referenceName, queue);

		//向线程池watchExecutor提交一个任务。在该任务里启动检测。
    watchExecutor.execute(new Runnable() {
      @Override public void run() {
        ensureGone(reference, watchStartNanoTime);
      }
    });
  }
  
  
  void ensureGone(KeyedWeakReference reference, long watchStartNanoTime) {
   	long gcStartNanoTime = System.nanoTime();

    long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);
    //移出引用队列里的对象
    removeWeaklyReachableReferences();
    if (gone(reference) || debuggerControl.isDebuggerAttached()) {
      return;
    }
    //启动垃圾回收
    gcTrigger.runGc();
    //移出引用队列里的对象
    removeWeaklyReachableReferences();
    //如果retainedKeys里边依然包含改若引用对象的key.说明该弱对象引用的对象泄露了。
    if (!gone(reference)) {
      long startDumpHeap = System.nanoTime();
      long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);
			//生成Dump文件
      File heapDumpFile = heapDumper.dumpHeap();

      if (heapDumpFile == null) {
        // Could not dump the heap, abort.
        return;
      }
      long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
      heapdumpListener.analyze(
          new HeapDump(heapDumpFile, reference.key, reference.name, watchDurationMs, gcDurationMs,
              heapDumpDurationMs));
    }
  }
  
  //移出如引用对象
   private void removeWeaklyReachableReferences() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    //垃圾回收启动之前，一旦弱引用对象引用的对象变得弱引用时，弱引用对象就会加入到它内部的队列里。
    KeyedWeakReference ref;
    //遍历队列，如果队列里的元素等于当前的对象，说明当前对象被回收了，就从retainedKeys里移出弱引用对象对一个的key.
    while ((ref = (KeyedWeakReference) queue.poll()) != null) {
      retainedKeys.remove(ref.key);
    }
  }
  
  //判断当前的reference对象对象的Key是否包含在retainedKeys，如果在就说明reference没有被垃圾回收
   private boolean gone(KeyedWeakReference reference) {
    return !retainedKeys.contains(reference.key);
  }
```

