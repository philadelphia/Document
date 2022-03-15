EventBus 3.0源码分析

## 简介

[EvenntBus](https://github.com/greenrobot/EventBus)  是一个Android开发中的用于事件分发的开源库，它的工作核心是发布/订阅者者模式。它可以利用很少的代码，来实现多组件间通信。android的组件间通信，我们不由得会想到handler消息机制和广播机制，通过它们也可以进行通信，但是使用它们进行通信，代码量多，组件间容易产生耦合引用。关于EventBus的工作模式，这里引用一张官方图帮助理解。
<img src="/Users/zhangtao/Desktop/EventBus-Publish-Subscribe.png" alt="EventBus-Publish-Subscribe" style="zoom:75%;" />

为什么会选择使用EventBus来做通信？

- 简化了组件间交流的方式

- 对事件通信双方进行解耦

- 可以灵活方便的指定工作线程，通过ThreadMode

- 速度快，性能好

- 库比较小，60k左右，对包大小无影响

- 使用这个库的app多，有权威性

- 功能多，使用方便

  

EventBus的使用也非常简单，其中三个重要的角色

1:Publisher 事件发布者

2:Subscriber 事件订阅者

3:Event 事件

Publisher post 事件后，Subscriber会自动收到事件(订阅方法会被主动调用，并将事件传递过来)。

## 使用

```
 //倒入gradle 依赖
 implementation 'org.greenrobot:eventbus:3.3.1'

```

1:定义事件类型

```
public static class MessageEvent { /* Additional fields if needed */ }
```

2:在需要订阅事件的模块中注册EventBus，页面销毁时注意注销

     @Override
     protected void onStart() {
       	super.onStart();
        EventBus.getDefault().register(this);    
     }
    
    @Override
    protected void onDestroy() {
        super.onDestroy();
        EventBus.getDefault().unregister(this);
    }
3:注册需要接受的事件类型 //注意同一种事件类型不能重复注册。不然会崩溃，且订阅方法必须是public类型的。

```
@Subscribe(threadMode = ThreadMode.MAIN)  
public void onMessageEvent(MessageEvent event) {
    // Do something
}
```

4.发送事件

```
EventBus.getDefault().post(new MessageEvent());
这是步骤3种的方法就会收到MessageEvent事件的回调
```



## 源码分析

EventBus 主类中只有不到600行代码，非常精简。EventBus使用了对外提供了单例模型，内部构建使用了Build模式。

### 1 register

```java
public void register(Object subscriber) {
    if (AndroidDependenciesDetector.isAndroidSDKAvailable() && 	!AndroidDependenciesDetector.areAndroidComponentsAvailable()) {
        // Crash if the user (developer) has not imported the Android compatibility library.
        throw new RuntimeException("It looks like you are using EventBus on Android, " +
                "make sure to add the \"eventbus\" Android library to your dependencies.");
    }

		//1从这里开始看，获取调用者的类对象。
    Class<?> subscriberClass = subscriber.getClass();
    //2 找到订阅类中的订阅方法
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    
    //3遍历订阅方法，将订阅者和其中的订阅方法绑定。
   	synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```

#### Step1

看下步骤2中的subscriberMethodFinder的findSubscriberMethods方法

```java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {	
		//1首先从缓存map中拿到订阅类的订阅方法列表，使用了缓存提高性能，nice，不出所料METHOD_CACHE的类型是Map<Class<?>, List<SubscriberMethod> >
    List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
    //2 如果不为空，说明之前该类曾经注册过，该类的新对象不必重新做绑定了，因为此时的操作是类层面的
    if (subscriberMethods != null) {
        return subscriberMethods;
    }
		//如果subscriberMethods 为null,说明该类是第一次注册，需要将其中的接收方法保存起来，
  	//ignoreGeneratedIndex 默认为false
    if (ignoreGeneratedIndex) {
        subscriberMethods = findUsingReflection(subscriberClass);
    } else {
        subscriberMethods = findUsingInfo(subscriberClass);
    }
    //如果subscriberMethods为null，说明当前类对象没有生命订阅方法，抛出异常
    if (subscriberMethods.isEmpty()) {
        throw new EventBusException("Subscriber " + subscriberClass
                + " and its super classes have no public methods with the @Subscribe annotation");
    } else {
      	//将当前注册类和其中的注册方法保存起来
        METHOD_CACHE.put(subscriberClass, subscriberMethods);
        return subscriberMethods;
    }
}
```

#### Step2

从步骤2中找出类的注册方法列表，然后遍历列表，调用下面的方法，将类对象和注册方法绑定。

```
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
		//1 找到订阅方法的事件类型，即发送事件的MessageEvent.class
    Class<?> eventType = subscriberMethod.eventType;
  
  	//2 将订阅者类对象和订阅事件绑定成一个对象
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    //subscriptionsByEventType 这个集合肯定是用来放置同一事件类型的订阅集合的，因为一个事件可能会有多个订阅的。
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions == null) {
        subscriptions = new CopyOnWriteArrayList<>();
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
    		//如果一个订阅者多次订阅了一个事件（@Subscribe注解的方法的参数是同一类型），抛出异常
        if (subscriptions.contains(newSubscription)) {
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                    + eventType);
        }
    }

		//3 按照订阅方法中@Subscribe中的priority参数进行排序，默认为最低优先级0。subscriptions种的对象按优先级排序，收到事件后就会			按优先级进行回调
    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            subscriptions.add(i, newSubscription);
            break;
        }
    }

		// typesBySubscriber类型Map<Object, List<Class<?>>>,Key 为订阅者，value为订阅者中的订阅方法，用来记录每个订阅者内部都订阅了哪些事件类型
    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        typesBySubscriber.put(subscriber, subscribedEvents);
    }
    subscribedEvents.add(eventType);

	//粘性事件相关
    if (subscriberMethod.sticky) {
        if (eventInheritance) {
            // Existing sticky events of all subclasses of eventType have to be considered.
            // Note: Iterating over all events may be inefficient with lots of sticky events,
            // thus data structure should be changed to allow a more efficient lookup
            // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                Class<?> candidateEventType = entry.getKey();
                if (eventType.isAssignableFrom(candidateEventType)) {
                    Object stickyEvent = entry.getValue();
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```



### 2 post

```
/** Posts the given event to the event bus. */
public void post(Object event) {
		//1首先获取当前线程的工作状态
    PostingThreadState postingState = currentPostingThreadState.get();
    //2获取当前线程的任务队列
    List<Object> eventQueue = postingState.eventQueue;
    //3 将事件加入到事件队列
    eventQueue.add(event);
	 //4 如果当前线程的工作状态没有正在发送事件 
    if (!postingState.isPosting) {
    		//标记postingState 的是否是主线程，并将工作状态isPosting 设为true
        postingState.isMainThread = isMainThread();
        postingState.isPosting = true;
        if (postingState.canceled) {
            throw new EventBusException("Internal error. Abort state was not reset");
        }
        try {
        		//遍历任务队列，发送事件
            while (!eventQueue.isEmpty()) {
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}


/**
* 发送事件
* @param event 事件
* @param postingState 当前线程相关的配置
*/
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        //如果使用继承事件的父类/接口，比如你发送了MessageEvent 事件，如果该事件继承了BaseEvent和Ievent接口,那么当你发送						MessageEvent 事件时，系统也会发送BaseEvent和Ieven事件
        if (eventInheritance) {
        		//遍历父类，将事件的父类/接口统统加入到eventTypes中
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                //遍历eventTypes，依次发送调用事件
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
           
        } else {
        		//不实用事件继承模型，直接发送该事件
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        if (!subscriptionFound) {
        		//如果该事件没有订阅者抛出异常
            if (logNoSubscriberMessages) {
                logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
            }
            //EventBus 内部也适用EventBus 发送了一个异常事件
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));
            }
        }
    }
    
```

#### postSingleEventForEventType

```
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
    CopyOnWriteArrayList<Subscription> subscriptions;
    synchronized (this) {
    	//1获取该事件的所有订阅关系列表
        subscriptions = subscriptionsByEventType.get(eventClass);
    }
    //2 遍历订阅关系列表，依次将事件发送到订阅者
    if (subscriptions != null && !subscriptions.isEmpty()) {
        for (Subscription subscription : subscriptions) {
            postingState.event = event;
            postingState.subscription = subscription;
            boolean aborted;
            try {
            		//将事件发送到订阅者
                postToSubscription(subscription, event, postingState.isMainThread);
                aborted = postingState.canceled;
            } finally {
                postingState.event = null;
                postingState.subscription = null;
                postingState.canceled = false;
            }
            if (aborted) {
                break;
            }
        }
        return true;
    }
    return false;
}
```

#### postToSubscription

这里就比较关键了，最终到了事件分发的地方了。

```
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
		//1 首先判断订阅关系中订阅方法的线程，就是声明线程时使用@Subcribe注解时传入的threadMode字段的值
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING: //直接发送事件
            invokeSubscriber(subscription, event);
            break;
        case MAIN:  //在主线程相应事件
       			 //事件发出线程是否是主线程
            if (isMainThread) { 是，直接发送
                invokeSubscriber(subscription, event);
            } else {不是通过mainThreadPoster发送
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case MAIN_ORDERED:
            if (mainThreadPoster != null) {
                mainThreadPoster.enqueue(subscription, event);
            } else {
                // temporary: technically not correct as poster not decoupled from subscriber
                invokeSubscriber(subscription, event);
            }
            break;
        case BACKGROUND: //在后台线程相应事件
        		 //事件发出线程是否是主线程
            if (isMainThread) {主线程发送事件，backgroundPoster转发
                backgroundPoster.enqueue(subscription, event);
            } else { 非主线程发送事件，直接发送
                invokeSubscriber(subscription, event);
            }
            break;
        case ASYNC: //异步线程相应事件
        		//通过asyncPoster发送事件
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```



####  invokeSubscriber

```
void invokeSubscriber(Subscription subscription, Object event) {
    try {
    	//非常暴力，直接通过回调调用订阅者中的订阅方法
        subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
    } catch (InvocationTargetException e) {
        handleSubscriberException(subscription, event, e.getCause());
    } catch (IllegalAccessException e) {
        throw new IllegalStateException("Unexpected exception", e);
    }
}
```



在postToSubscription中有三个重要的角色mainThreadPoster，backgroundPoster，asyncPoster

其中mainThreadPoster的类型是HandlerPoster。其实就是Handler。调用其enqueu()方法

而backgroundPoster和asyncPoster 本质都是Runnable

### 3 unregister 

解绑方法就简单多了，重要的是就把register里提到的2个重要的几何中删除订阅者

```
/** Unregisters the given subscriber from all event classes. */
public synchronized void unregister(Object subscriber) {
	//1 从typesBySubscriber找到订阅者所订阅的事件类型列表
    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
    if (subscribedTypes != null) {
    	//2 遍历列表，依次解绑订阅者和事件类型。应该是从post分析里的订阅事件集合subscriptionsByEventType里移除对应事件类型的该订阅者
        for (Class<?> eventType : subscribedTypes) {
            unsubscribeByEventType(subscriber, eventType);
        }
        //3移除订阅者
        typesBySubscriber.remove(subscriber);
    } else {
        logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
    }
}

/**
* 解绑订阅者和事件类型
* @param subscriber 订阅者
* @param eventType  订阅的事件类型
*/
 private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
				//从subscriptionsByEventType里获取该订阅事件的订阅者集合。
        List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions != null) {
            int size = subscriptions.size();
            for (int i = 0; i < size; i++) {
            		//遍历集合，获取所有的订阅关系
                Subscription subscription = subscriptions.get(i);
                if (subscription.subscriber == subscriber) {
                    subscription.active = false;
                    subscriptions.remove(i);
                    //重要。不然会抛出ConcurrentModifyException
                    i--;
                    size--;
                }
            }
        }
    }

```



### 4 Subscribe注解

```
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface Subscribe {
    ThreadMode threadMode() default ThreadMode.POSTING;

    /**
     * If true, delivers the most recent sticky event (posted with
     * {@link EventBus#postSticky(Object)}) to this subscriber (if event available).
     */
    boolean sticky() default false;

    /** Subscriber priority to influence the order of event delivery.
     * Within the same delivery thread ({@link ThreadMode}), higher priority subscribers will receive events before
     * others with a lower priority. The default priority is 0. Note: the priority does *NOT* affect the order of
     * delivery among subscribers with different {@link ThreadMode}s! */
    int priority() default 0;
}
```

### 5 ThreadMode

```
//定义事件回调方法工作线程的类
public enum ThreadMode {
    /**
     *直接在发送事件的线程里调用Subscriber，这个是默认的设置，事件交付开销最小，因为它避免了线程切换。因此它是那种很快完成的单任务			 *默认的的线程工作模型。使用该模型的事件必须很快完成，因为当发布线程是主线程时，它可能阻塞主线程。
     /
    POSTING,

    /**
     * 在Android平台，订阅者将会在Android的主线程调用。如果发布线程时主线程，订阅方法将会被直接调用。进而阻赛发布线程，如果发布线			 * 程不是主线程。事件将会排队等待分发。使用这种模式的订阅者必须快速完成任务，避免阻赛主线程。非Android平台和Posting一样
     */
    MAIN,

    /**
     * 在Android平台，订阅者将会在Android的主线程调用。不同于MAIN，事件将会有序分发。确保了post调用时非阻赛的。
     */
    MAIN_ORDERED,

    /** 
     * 在Android平台，订阅者将会在后台线程被调用，如果发布线程不是主线程，订阅者将会被直接调用，如果发布线程时主线程，那么EventBus 			* 使用后台线程，进而有序分发所有事件，使用此模式的Subscribers应该快速完成任务以免阻赛后台线程。非Android平台，总是用后台线程          		 * 相应事件 
     */
    BACKGROUND,

    /**
     * 订阅者将会在单独的线程被调用，总是独立于发布线程和主线程。发布事件从不会等待使用这种模式的订阅方法。如果订阅方法执行耗时任务，		 * 则应该使用此模式。比如；网络访问。避免同时触发大量的长时间运行的异步订阅方法，从而限制并发的线程数量。EventBus 使用线程池来   		 * 高效的服用已完成异步订阅通知的线程
     */
    ASYNC
}
```

### 6 EventBus2.0和EventBus3.0的区别？

这是在面试过程中，面试官最常问的一个问题。

EventBus2.0和3.0最大的区别有两点：

1.EventBus2.0中我们在书写订阅方法时的名字必须是onEvent开头，然后通过命名不同来区别不同的线程模式。例如对应posting则命名为onEvent()，onEventMainThread()则对应main等。而3.0则可以用任何名字作为方法名称，只需要在方法名的前面用@Subscribe注解来进行注释，然后使用threadMode来设置在哪里线程中接收事件和处理事件

2.EventBus2.0使用的是反射的方式来查找所有的订阅方法，而3.0则是在编译时通过注解处理器的方式来查找所有的订阅方法。性能上来说，3.0比2.0要高的多。