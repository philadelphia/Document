# 						JAVA 数据结构-队列

队列是线性表的一种，具有先进先出(FIFO)的特性,只能在队列尾部插入元素，头部删除

```
public interface Queue<E> extends Collection<E> {
    /**
     * 向队列中插入元素，如果可能立即插入而不违反容量限制
     * 成功返回true，失败抛出异常
     * Inserts the specified element into this queue if it is possible to do so
     * immediately without violating capacity restrictions, returning
     * {@code true} upon success and throwing an {@code IllegalStateException}
     * if no space is currently available.
     *
     * @param e the element to add
     * @return {@code true} (as specified by {@link Collection#add})
     * @throws IllegalStateException if the element cannot be added at this
     *         time due to capacity restrictions
     * @throws ClassCastException if the class of the specified element
     *         prevents it from being added to this queue
     * @throws NullPointerException if the specified element is null and
     *         this queue does not permit null elements
     * @throws IllegalArgumentException if some property of this element
     *         prevents it from being added to this queue
     */
    boolean add(E e);

    /**
     * 向队列中插入元素，如果可能立即插入而不违反容量限制
     * 当使用容量受限制的队列时，改方法一般来说优于add方法，add当插入失败时会抛出异常
     * Inserts the specified element into this queue if it is possible to do
     * so immediately without violating capacity restrictions.
     * When using a capacity-restricted queue, this method is generally
     * preferable to {@link #add}, which can fail to insert an element only
     * by throwing an exception.
     *
     * @param e the element to add
     * @return {@code true} if the element was added to this queue, else
     *         {@code false}
     * @throws ClassCastException if the class of the specified element
     *         prevents it from being added to this queue
     * @throws NullPointerException if the specified element is null and
     *         this queue does not permit null elements
     * @throws IllegalArgumentException if some property of this element
     *         prevents it from being added to this queue
     */
    boolean offer(E e);

    /**
     * 取出并删除对头元素. 与方法poll不同的是，如果队列为空，该方法抛出NoSuchElementException异常
     * @return the head of this queue
     * @throws NoSuchElementException if this queue is empty
     */
    E remove();

    /**
     * 取出并删除对头元素
     *如果队列为空，返回null 
     *
     */
    E poll();

    /**
     * 获取但是不删除队头元素, 改方法与peek不同的是，如果队列为空，只会抛出一个NoSuchElementException异			* 常 
     * @return the head of this queue
     * @throws NoSuchElementException if this queue is empty
     */
    E element();

    /**
     * 获取并不删除队头元素，如果队列为null， 返回null, 
     * or returns {@code null} if this queue is empty.
     * @return the head of this queue, or {@code null} if this queue is empty
     */
    E peek();
}
```

generally speaking ，this interface provides two methods for inserting element add and offer.

two method to retrieve and remove element,this differences between them is that : remove can throw an expection when remove failed and poll will retrun null if the queue is empty.

two method to retrieve but not delele element, this differences between them is that : element can throw an expection when remove failed and peek will retrun null if the queue is empty.



直接子类是AbstractQueue,AbstractQueue又定义了一个addAll方法和clear方法，用来清空队列

而AbstractQueue又有四个直接子类:

ArrayBlockingQueue

```java
ArrayBlockingQueue
ConcurrentLinkedQueue
DelayQueue
LinkedBlockingQueue
LinkedBlockingDeque
ProirityBlockingQueue
ProirityQueue
SynchronousQueue
LinkedTransferQueue
```

```
BlockingQueue 接口继承了Queue，提供了阻塞的功能，所有的方法都添加了同步实现 
```