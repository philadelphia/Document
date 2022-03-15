阻塞队列，也就是 BlockingQueue，它是一个接口，如代码所示：



​    public interface BlockingQueue<E> extends Queue<E>{...}



BlockingQueue 继承了 Queue 接口，是队列的一种。Queue 和 BlockingQueue 都是在 Java 5 中加入的。



BlockingQueue 是线程安全的，我们在很多场景下都可以利用线程安全的队列来优雅地解决我们业务自身的线程安全问题。比如说，使用生产者/消费者模式的时候，我们生产者只需要往队列里添加元素，而消费者只需要从队列里取出它们就可以了

既然队列本身是线程安全的，队列可以安全地从一个线程向另外一个线程传递数据，所以我们的生产者/消费者直接使用线程安全的队列就可以，而不需要自己去考虑更多的线程安全问题。这也就意味着，考虑锁等线程安全问题的重任从“你”转移到了“队列”上，降低了我们开发的难度和工作量。

Java 提供的线程安全的队列（也称为并发队列）分为阻塞队列和非阻塞队列两大类。

![](file:///Users/meiliwu/Documents/Gridea/post-images/1593189553251.png)

可以看出Queue接口提供了以上方法。

而BlockingQueue又新增了get()和put()方法。

把 BlockingQueue 中最常用的和添加、删除相关的 8 个方法列出来，并且把它们分为三组，每组方法都和添加、移除元素相关。



1：添加元素 



​    boolean add(E e) ;

​    boolean offer(E e);

​    void put(E e);



2:删除元素

   

​    E remove() ;

​    E get();



3.获取队列头部元素

​    

​    E element();

​    E peek();

​    

这三组方法由于功能很类似，所以比较容易混淆。它们的区别仅在于特殊情况：当队列满了无法添加元素，或者是队列空了无法移除元素时，不同组的方法对于这种特殊情况会有不同的处理方式：



1：抛出异常：add、remove、element

2：返回结果但不抛出异常：offer、poll、peek

3：阻塞：put、take

\## 第一组：add、remove、element

\### add()

方法向队列中添加一个元素，如果队列没有满，这个时候添加成功，返回true

如果此时队列已经满了，则会这届抛出一个异常



\### remove

remove方法从队列中删除头部元素，

如果队列不为空，则将队列头部元素从队列中移出并返回队改元素

如果队列为0，则直接抛出一个NoSuchElementException 异常



\### element方法

该方法返回队列头部元素

如果队列不为空，则直接返回队列头部元素。如果队列为空，则直接抛出NoSuchElementException

改方法和remove方法的区别就是该方法只会去队列的头部元素而不会删除头部元素



\## 第二组：offer、poll、peek



第二组方法相比于第一组而言要友好一些，当发现队列满了无法添加，或者队列为空无法删除的时候，第二组方法会给一个提示，而不是抛出一个异常。



\### offer 方法

offer 方法用来插入一个元素，并用返回值来提示插入是否成功。如果添加成功会返回 true，而如果队列已经满了，此时继续调用 offer 方法的话，它不会抛出异常，只会返回一个错误提示：false



\### poll 方法

poll 方法和第一组的 remove 方法是对应的，作用也是移除并返回队列的头节点。但是如果当队列里面是空的，没有任何东西可以移除的时候，便会返回 null 作为提示。正因如此，我们是不允许往队列中插入 null 的，否则我们没有办法区分返回的 null 是一个提示还是一个真正的元素



\### peek 方法

peek 方法和第一组的 element 方法是对应的，意思是返回队列的头元素但并不删除。如果队列里面是空的，它便会返回 null 作为提示



带超时时间的 offer 和 poll



offer 和 poll 都有带超时时间的重载方法。

​    offer(E e, long timeout, TimeUnit unit)



它有三个参数，分别是元素、超时时长和时间单位。通常情况下，这个方法会插入成功并返回 true；如果队列满了导致插入不成功，在调用带超时时间重载方法的 offer 的时候，则会等待指定的超时时间，如果时间到了依然没有插入成功，就会返回 false



​    poll(long timeout, TimeUnit unit)



带时间参数的 poll 方法和 offer 类似：如果能够移除，便会立刻返回这个节点的内容；如果队列是空的就会进行等待，等待时间正是我们指定的时间，直到超时时间到了，如果队列里依然没有元素可供移除，便会返回 null 作为提示



\## 第三组：put、take

这两个方法是阻塞队列特有的，Queue接口并不存在这个方法的声明。

这两个方法的特定是在无法完成操作是会阻塞线程、



\### put 方法

put 方法的作用是插入元素。通常在队列没满的时候是正常的插入，但是如果队列已满就无法继续插入，这时它既不会立刻返回 false 也不会抛出异常，而是让插入的线程陷入阻塞状态，直到队列里有了空闲空间，此时队列就会让之前的线程解除阻塞状态，并把刚才那个元素添加进去



\### take 方法

take 方法的作用是获取并移除队列的头结点。通常在队列里有数据的时候会正常取出数据并删除；但是如果执行 take 的时候队列里无数据，则阻塞，直到队列里有数据；一旦队列里有数据了，就会立刻解除阻塞状态，并且取到数据



总和以上的分析，可以将以上分方法的区别用一个表格表示

![](file:///Users/meiliwu/Documents/Gridea/post-images/1593191082633.png)