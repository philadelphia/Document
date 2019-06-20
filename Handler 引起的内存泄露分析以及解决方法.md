# Handler 引起的内存泄露分析以及解决方法

Handler是Android系统提供的一种在子线程更新UI的机制，但是使用不当会导致memory leak。严重的话可能导致OOM

Java语言的垃圾回收机制采用了可达性分析来判断一个对象是否还有存在的必要性，如无必要就回收该对象引用的内存区域，

```java
Handler handler ；
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    handler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
        }
    };

}


```

然后在其他地方来发送一个延迟消息

```
handler.postDelayed(new Runnable() {
    @Override
    public void run() {
        
    }
}, 500);
```

我们一般使用Handler就是扎样，但是这样会当Activity销毁后会导致memory leak.

原因就是activity销毁了，但是以为我们的Handler对象是一个内部类，因为内部类会持有外部类的一个引用。所以当activity销毁了，但是因为Handler还持有改Activity的引用，导致GC启动后，可达性分析发现该Activity对象还有其他引用。所以无法销毁改Activity，

但是handler仅仅是Activity的一个内存对象。及时他引用了Activity,他们之间也只是循环引用而已。而循环引用则不影响GC回收内存。

其实真正的原因是Handler调用postDelayed发送一个延迟消息时：

```java
public final boolean postDelayed(Runnable r, long delayMillis)
{
    return sendMessageDelayed(getPostMessage(r), delayMillis);
}
```

而sendMessageDelayed的实现是

```java
public final boolean sendMessageDelayed(Message msg, long delayMillis)
{
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
```

再往下看

```java
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}
```

最终是将该消息加入到消息队列中。

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

可以看到，在enqueueMessage的实现中。将msg.target = this;

就是讲改Handler对象赋值给了message的target对象。所以message对象就引用了Handler对象''

进而messageQueue对象就引用了Handler对象。此次逐渐明朗。就是messagequeue———message———

Handler——Activity。

所以我们可以在任一环节做文章即可避免Handler持有Activity对象导致的内存泄露问题。

我们可以在Activity销毁时将任务队列清空，或者 在Activity 销毁时将Handler对象销毁。

总之，就是在任一环节将该引用链条切换就好了，这样GC就可以销毁Activity对象了。

此时还是没有触及到问题的核心，就是为什么messageQueue为什么会持有message对象进而持有Handler对象，导致Activity销毁时还有其他引用。为什么Activity销毁时MessageQueue不销毁呢，这才是问题的核心，如果messageQueue销毁了啥问题也没有了。当然我们也可以在Activity销毁时手动销毁messageQueue对象。这样也可以避免内存泄露。

从这我们可以看出messagequeue的生命周期比Activity长了。所以才导致这些问题。

其实熟悉Handler机制的话就会明白背后的原因了



```java
 final Looper mLooper;
 final MessageQueue mQueue;

public Handler() {
    this(null, false);
}

public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

从构造方法我们可以看出，无参的构造方法最终调用了两参的构造方法。

```
mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
```

这几行代码才是重中之重。

首先调用Looper.myLooper()方法。如果looper为null，说明没有调用looper.prepare()方法。从抛出的运行时异常可以看出来。(ps:所以在子线程使用handler时，第一就是要调用Looper.prepare方法)

looper不为空话话，将looper复制个Handler的looper对象，然后将looper的queue对象赋值给handler的queue对象。

可以说Handler的looper字段和queue字段都是来着looper对象的。

可以看出我们在Handler里发送的消息最终发送到了handler的queue对象所执行的内存区域，而这片内存区域也是Looper对象的queue对象所指向的。所以说该queue对象里所有的message对象都收到Looper对象的queue对象的管理。

真正的大boss来了，都是Looper搞鬼。

因为我们是在主线程中初始化的Handler。所以Handler引用的looper对象是在主线程中创建的。



在代码ActivityThread.main()中：

```java
public static void main(String[] args) {
        ....

        //创建Looper和MessageQueue对象，用于处理主线程的消息
        Looper.prepareMainLooper();

        //创建ActivityThread对象
        ActivityThread thread = new ActivityThread(); 

        //建立Binder通道 (创建新线程)
        thread.attach(false);

        Looper.loop(); //消息循环运行
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```



```
 Looper.prepareMainLooper();
 
  public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
```

在prepareMainLooper方法中首先调用了prepare方法，这就是为什么我们在主线程使用Handler时不需要自己手动调动looper的prepare方法的原因。

```java
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```

在prepare方法中首先从sThreadLocal对象中取出looper对象。如果不为null.说明已经初始化过了，直接抛出异常。

没有初始化的话直接初始化然后放到sThreadLocal中。sThreadLocal是一个ThreadLocal类型。持有线程的私有数据。

此时，真相大白了。主线程的ThreadLocal——>looper——>messagequue——>message——>handler——>Acitivity



因为APP在活动中，所以主线程一直存在。looper一直存在，messageQueue一直存在。所以当我们发送了延迟消息时，而此时Activity销毁的话。自然会引起内存泄露的。

解决方法也很明了了。既然我们不能再looper层面做文章，就只能在handler和message层面做文章了。在Activity销毁时 将Handler手动置为null,或者将messagequeue 清空，或者将Handler设置为静态内部类。然后内部通过若引用持有Activity对象。总之就是要让Handler和message改放手时就放手

