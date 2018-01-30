

## Function ##
Function 是Action和Func的父接口。

	/**
	 * All Func and Action interfaces extend from this.
	 * <p>
	 * Marker interface to allow instanceof checks.
	 */
	public interface Function {
	
	}
    

##  Action ##

	/**
     * All Action interfaces extend from this.
     * <p>
     * Marker interface to allow instanceof checks.
     */
    public interface Action extends Function {
    
    }

 Action0 是 RxJava 的一个接口，它只有一个方法 call()，这个方法是无参无返回值的；

Action 接口有以下子接口，
Action1,Action2------Action9,ActionN

    public interface Action1<T> extends Action {
    	void call(T t);
	}
	
	public interface Action2<T1, T2> extends Action {
		void call(T1 t1, T2 t2);
	}
	
	public interface ActionN extends Action {
    	void call(Object... args);
	}

## Func ##
	Func接口几次Function接口和Callable接口，并复写Callable接口的call方法。
	Func0接口没有参数，只有返回值，
	Func1有一个参数，
	Func2有两个参数。
	.
	.
	.
	Func9有9个参数，
	以此类推，FuncN有N个参数


	/**
	 * Represents a function with zero arguments.
	 */
	public interface Func0<R> extends Function, Callable<R> {
		@Override
		R call();
	}
	
	/**
	 * Represents a function with one argument.
	 */
	public interface Func1<T, R> extends Function {
    	R call(T t);
	}
	
	/**
	 * A vector-argument action.
     */
	public interface Func9<T1, T2, T3, T4, T5, T6, T7, T8, T9, R> extends Function {
    	R call(T1 t1, T2 t2, T3 t3, T4 t4, T5 t5, T6 t6, T7 t7, T8 t8, T9 t9);
	}


在创建Observable以后，我们可以通过Observable.subscribe(final Action1<? super T> onNext, final Action1<Throwable> onError, final Action0 onCompleted)
	
如果我们只传递onNext，onError和onComplete会使用系统默认的。
	
	public final Subscription subscribe(final Action1<? super T> onNext) {
        if (onNext == null) {
            throw new IllegalArgumentException("onNext can not be null");
        }

        Action1<Throwable> onError = InternalObservableUtils.ERROR_NOT_IMPLEMENTED;
        Action0 onCompleted = Actions.empty();
        return subscribe(new ActionSubscriber<T>(onNext, onError, onCompleted));
    }

	 public final Subscription subscribe(final Action1<? super T> onNext, final Action1<Throwable> onError) {
        if (onNext == null) {
            throw new IllegalArgumentException("onNext can not be null");
        }
        if (onError == null) {
            throw new IllegalArgumentException("onError can not be null");
        }

        Action0 onCompleted = Actions.empty();
        return subscribe(new ActionSubscriber<T>(onNext, onError, onCompleted));
    }

	public final Subscription subscribe(final Action1<? super T> onNext, final Action1<Throwable> onError, final Action0 onCompleted) {
        if (onNext == null) {
            throw new IllegalArgumentException("onNext can not be null");
        }
        if (onError == null) {
            throw new IllegalArgumentException("onError can not be null");
        }
        if (onCompleted == null) {
            throw new IllegalArgumentException("onComplete can not be null");
        }

        return subscribe(new ActionSubscriber<T>(onNext, onError, onCompleted));
    }

	
最终系统会将onNext,onError，onComplete，组合成一个ActionSubscriber，
	
	new ActionSubscriber<T>(onNext, onError, onCompleted)
	/**
	 * A Subscriber that forwards the onXXX method calls to callbacks.
	 * @param <T> the value type
	 */
		public final class ActionSubscriber<T> extends Subscriber<T> {
	
	    final Action1<? super T> onNext;
	    final Action1<Throwable> onError;
	    final Action0 onCompleted;
	
	    public ActionSubscriber(Action1<? super T> onNext, Action1<Throwable> onError, Action0 onCompleted) {
	        this.onNext = onNext;
	        this.onError = onError;
	        this.onCompleted = onCompleted;
	    }
	
	    @Override
	    public void onNext(T t) {
	        onNext.call(t);
	    }
	
	    @Override
	    public void onError(Throwable e) {
	        onError.call(e);
	    }
	
	    @Override
	    public void onCompleted() {
	        onCompleted.call();
	    }
	}
		
## Observer ##
	
	public interface Observer<T> {
    	void onCompleted();

    	void onError(Throwable e);

    	void onNext(T t);

}

## Subscriber ##
Subscriber实现了Subscription接口，复写了其`unsubscribe`和`isUnsubscribed`方法。
在`Observer`的基础上增加了`onStart`，`request`，`addToRequested`，`setProducer`方法。


	public abstract class Subscriber<T> implements Observer<T>, Subscription{
    
	    // represents requested not set yet
	    private static final Long NOT_SET = Long.MIN_VALUE;
	
	    private final SubscriptionList subscriptions;
	    private final Subscriber<?> subscriber;
	    /* protected by `this` */
	    private Producer producer;
	    /* protected by `this` */
	    private long requested = NOT_SET; // default to not set
	
	    protected Subscriber() {
	        this(null, false);
	    }
	
	
	    protected Subscriber(Subscriber<?> subscriber) {
	        this(subscriber, true);
	    }
	
	   
	    protected Subscriber(Subscriber<?> subscriber, boolean shareSubscriptions) {
	        this.subscriber = subscriber;
	        this.subscriptions = shareSubscriptions && subscriber != null ? subscriber.subscriptions : new SubscriptionList();
	    }
	
	
	    public final void add(Subscription s) {
	        subscriptions.add(s);
	    }
	
	    @Override
	    public final void unsubscribe() {
	        subscriptions.unsubscribe();
	    }

    
	    @Override
	    public final boolean isUnsubscribed() {
	        return subscriptions.isUnsubscribed();
	    }
	
	    
	    public void onStart() {
	        // do nothing by default
	    }
	    
	    
	    protected final void request(long n) {
	        if (n < 0) {
	            throw new IllegalArgumentException("number requested cannot be negative: " + n);
	        } 
	        
	        // if producer is set then we will request from it
	        // otherwise we increase the requested count by n
	        Producer producerToRequestFrom = null;
	        synchronized (this) {
	            if (producer != null) {
	                producerToRequestFrom = producer;
	            } else {
	                addToRequested(n);
	                return;
	            }
	        }
	        // after releasing lock (we should not make requests holding a lock)
	        producerToRequestFrom.request(n);
	    }

	    private void addToRequested(long n) {
	        if (requested == NOT_SET) {
	            requested = n;
	        } else { 
	            final long total = requested + n;
	            // check if overflow occurred
	            if (total < 0) {
	                requested = Long.MAX_VALUE;
	            } else {
	                requested = total;
	            }
	        }
	    }
	    
	  
	    public void setProducer(Producer p) {
	        long toRequest;
	        boolean passToSubscriber = false;
	        synchronized (this) {
	            toRequest = requested;
	            producer = p;
	            if (subscriber != null) {
	                // middle operator ... we pass through unless a request has been made
	                if (toRequest == NOT_SET) {
	                    // we pass through to the next producer as nothing has been requested
	                    passToSubscriber = true;
	                }
	            }
	        }
	        // do after releasing lock
	        if (passToSubscriber) {
	            subscriber.setProducer(producer);
	        } else {
	            // we execute the request with whatever has been requested (or Long.MAX_VALUE)
	            if (toRequest == NOT_SET) {
	                producer.request(Long.MAX_VALUE);
	            } else {
	                producer.request(toRequest);
	            }
	        }
	    }
	}

## Observable 与Observer建立绑定关系 ##
Observable.create().subscribe(new Observer)

	  public final Subscription subscribe(final Observer<? super T> observer) {
	        if (observer instanceof Subscriber) {
	            return subscribe((Subscriber<? super T>)observer);
	        }
	        return subscribe(new ObserverSubscriber<T>(observer));
	    }

如果传入的直接是一个Subscriber对象的话，直接调用subscribe（Subscriber observer）
否则调用subsribe(new ObserverSubscriber<T>(observer)),将observer构造成一个ObserverSubscriber对象传入。
其实两者都是一样的。因为`ObserverSubscriber` 也是`Subscriber`的子类。
## ObserverSubscriber ##
	
		/**
		 * Wraps an Observer and forwards the onXXX method calls to it.
		 * @param <T> the value type
		 */
		public final class ObserverSubscriber<T> extends Subscriber<T> {
	    final Observer<? super T> observer;
	
	    public ObserverSubscriber(Observer<? super T> observer) {
	        this.observer = observer;
	    }
	    
	    @Override
	    public void onNext(T t) {
	        observer.onNext(t);
	    }
	    
	    @Override
	    public void onError(Throwable e) {
	        observer.onError(e);
	    }
	    
	    @Override
	    public void onCompleted() {
	        observer.onCompleted();
	    }
	}
该类包裹了一个Observer对象。所有的实现其实是Observer对象来操作的。


		 public final Subscription subscribe(Subscriber<? super T> subscriber) {
		        return Observable.subscribe(subscriber, this);
		    }



	    static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
	     // validate and proceed
	        if (subscriber == null) {
	            throw new IllegalArgumentException("observer can not be null");
	        }
	        if (observable.onSubscribe == null) {
	            throw new IllegalStateException("onSubscribe function can not be null.");
	            /*
	             * the subscribe function can also be overridden but generally that's not the appropriate approach
	             * so I won't mention that in the exception
	             */
	        }
	        
	        // new Subscriber so onStart it
			先调用subscriber的onStart方法。
	        subscriber.onStart();
	        
	        /*
	         * See https://github.com/ReactiveX/RxJava/issues/216 for discussion on "Guideline 6.4: Protect calls
	         * to user code from within an Observer"
	         */
	        // if not already wrapped
	        if (!(subscriber instanceof SafeSubscriber)) {
	            // assign to `observer` so we return the protected version
	            subscriber = new SafeSubscriber<T>(subscriber);
	        }
	
	        // The code below is exactly the same an unsafeSubscribe but not used because it would 
	        // add a significant depth to already huge call stacks.
	        try {
	            // allow the hook to intercept and/or decorate
	            hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
	            return hook.onSubscribeReturn(subscriber);
	        } catch (Throwable e) {
	            // special handling for certain Throwable/Error/Exception types
	            Exceptions.throwIfFatal(e);
	            // in case the subscriber can't listen to exceptions anymore
	            if (subscriber.isUnsubscribed()) {
	                RxJavaPluginUtils.handleException(hook.onSubscribeError(e));
	            } else {
	                // if an unhandled error occurs executing the onSubscribe we will propagate it
	                try {
	                    subscriber.onError(hook.onSubscribeError(e));
	                } catch (Throwable e2) {
	                    Exceptions.throwIfFatal(e2);
	                    // if this happens it means the onError itself failed (perhaps an invalid function implementation)
	                    // so we are unable to propagate the error correctly and will just throw
	                    RuntimeException r = new OnErrorFailedException("Error occurred attempting to subscribe [" + e.getMessage() + "] and then again while trying to pass to onError.", e2);
	                    // TODO could the hook be the cause of the error in the on error handling.
	                    hook.onSubscribeError(r);
	                    // TODO why aren't we throwing the hook's return value.
	                    throw r;
	                }
	            }
	            return Subscriptions.unsubscribed();
	        }
	    }

该方法返回一个`Subscription`对象

## Subscription ##
	
		public interface Subscription {
	
	    /**
	     * Stops the receipt of notifications on the {@link Subscriber} that was registered when this Subscription
	     * was received.
	     * <p>
	     * This allows unregistering an {@link Subscriber} before it has finished receiving all events (i.e. before
	     * onCompleted is called).
	     */
	    void unsubscribe();
	
	    /**
	     * Indicates whether this {@code Subscription} is currently unsubscribed.
	     *
	     * @return {@code true} if this {@code Subscription} is currently unsubscribed, {@code false} otherwise
	     */
	    boolean isUnsubscribed();
	
	}

该接口有两个方法。