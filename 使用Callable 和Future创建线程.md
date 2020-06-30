## 使用Callable 和Future创建线程

前面已经说过，这种方式的本质上和其他创建线程的方式是一致的，只有这种创建线程的方式是可以回去线程的返回结果的。

传统的runnable接口的定义

```
public interface Runnable {
    /**
     * When an object implementing interface <code>Runnable</code> is used
     * to create a thread, starting the thread causes the object's
     * <code>run</code> method to be called in that separately executing
     * thread.
     * <p>
     * The general contract of the method <code>run</code> is that it may
     * take any action whatsoever.
     *
     * @see     java.lang.Thread#run()
     */
    public abstract void run();
}
```

runnable只提供了一个返回void的run方法，而且执行失败也不会抛出异常。

这就导致使用runnable方式创建的线程执行时我们无法获取到返回值，而且失败我们也无法获取到具体的失败信息。

而且提交到线程池的Runnable任务时无法取消的，这就意味着如果我们想要取消某个任务是不可能的，只能关闭整个线程池。

综合以上，java在JDK1.5退出了Callable和Future。

```
@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```

从Callable接口的定义可以看出其支持泛型，返回值类型为传入的泛型参数。而且支持抛出异常。

下面是Future接口的定义

```
public interface Future<V> {

    /**
     * 尝试去取消整个任务，可能还失败如果该任务已经完成或者已经被取消，或者由于其他原因不能被取消。
     * 取过取消任务成功，整个任务的cancel方法被调用时任务还没有开始。如果这个任务已经启动了，参数 
   	 *	  mayInterruptIfRunning 决定了是否给执行任务的线程发送中断信号。
   	 *	  这个方法返回后，后续的isDone调用总是返回true。
   	 *	  后续的isCancelled调用总是返回true，如果这个方法返回了true。
     */
    boolean cancel(boolean mayInterruptIfRunning);

    /**
     * Returns {@code true} if this task was cancelled before it completed
     * normally.
     *
     * @return {@code true} if this task was cancelled before it completed
     */
    boolean isCancelled();

    /**
     * Returns {@code true} if this task completed.
     *
     * Completion may be due to normal termination, an exception, or
     * cancellation -- in all of these cases, this method will return
     * {@code true}.
     *
     * @return {@code true} if this task completed
     */
    boolean isDone();

    /**
     * Waits if necessary for the computation to complete, and then
     * retrieves its result.
     *
     * @return the computed result
     * @throws CancellationException if the computation was cancelled
     * @throws ExecutionException if the computation threw an
     * exception
     * @throws InterruptedException if the current thread was interrupted
     * while waiting
     */
    V get() throws InterruptedException, ExecutionException;

    /**
     * Waits if necessary for at most the given time for the computation
     * to complete, and then retrieves its result, if available.
     *
     * @param timeout the maximum time to wait
     * @param unit the time unit of the timeout argument
     * @return the computed result
     * @throws CancellationException if the computation was cancelled
     * @throws ExecutionException if the computation threw an
     * exception
     * @throws InterruptedException if the current thread was interrupted
     * while waiting
     * @throws TimeoutException if the wait timed out
     */
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

Future接口提供了4个方法

```
boolean cancel(boolean mayInterruptIfRunning);
boolean isCancelled();
boolean isDone();
V get() throws InterruptedException, ExecutionException;
V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
```

### get

get方法用于获取Callable的执行结果，通常有5中情况发生

1. 任务已经正常结束，此时获取到Callable的返回结果。

2. 任务还没有结束，这是很有可能的，因为我么你通常将任务放到线程池中，有可能你调用get方法是，任务还在任务队列中没被执行呢，另外一种情况是任务已经执行了，但是可能需要很长时间才能返回，

   这两种情况下，调用get方法时自然是获取不到结果的，都会阻塞当前线程。知道任务完成返回结果。

3. 任务执行过程中被cancel，这个时候任务会抛出CancellationException异常

4. 任务执行过程中失败，这个时候任务会抛出ExecutionException异常。

5. 任务超时。get方法有个带有参数的重载方法。调用带有延迟参数的get方法后，如果在指定时间内任务执行完毕，返回结果，如果任务无法完成工作，直接抛出TimeoutException异常。

### isDone

用来判断任务是否执行完毕

如果任务完成了返回true。如果任务没有完成返回false。

需要注意的是该方法返回true不代表任务是成功执行了，只代表任务结束了。如果任务执行过程中被取消了。该方法依旧返回true。因为该任务确实执行完毕了，以后不会在被执行了。

## 用 FutureTask 来创建 Future

```
ExecutorService executorService = Executors.newCachedThreadPool();
Future<Integer> submit = executorService.submit(new Callable<Integer>() {
    @Override
    public Integer call() throws Exception {
        return null;
    }
});
```

除了使用线程池提交一个Callable任务会返回一个Future对象外，我们也可以使用FutureTask来获取Future类的结果

```
Callable<Integer> callable = new Callable<Integer>() {
    @Override
    public Integer call() throws Exception {
        return null;
    }
};
FutureTask<Integer> integerFutureTask = new FutureTask<>(callable);
new Thread(integerFutureTask).start();
```

典型用法是，把 Callable 实例当作 FutureTask 构造函数的参数，生成 FutureTask 的对象，然后把这个对象当作一个 Runnable 对象，这里使用了桥接的设计模式。放到线程池中或另起线程去执行，最后还可以通过 FutureTask 获取任务执行的结果。但是使用Future获取结果时需要注意，get方法在没有任务没有执行完毕时会阻塞调用者线程。

