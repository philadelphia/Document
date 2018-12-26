

动态绑定作为Java多态特性的一个重要实现

有一下三个类：

```java
public class Fruit {
    public static final String TAG = "Fruit";

    public Fruit() {
    }

    public void say() {
        Log.i(TAG, "say: i am fruit: ");
    }
}
```

```java
public class Apple  extends  Fruit{

    public void say(){
        Log.i(TAG, "say:  i am apple");
    }
}
```



```java

public class Orange extends Fruit {

    public void say(){
        Log.i(TAG, "say:  i am orange");
    }
}
```

 其中Apple和Orange类都继承Fruit类。

然后在使用的时候

```java
Fruit fruit = new Apple();
fruit.say();
```

我们都知道最后打印的结果是

```java
debug I/Fruit: say:  i am apple
```

很明显这就是Java多态特性的体现。

我们明明调用的是Fruit对象的方法。但是运行时却调用了Apple对象的方法，这是怎么实现的呢，这就涉及到了Java的动态绑定了，

这里确定两个概念：编译时类型与运行时类型

编译时类型就是指改对象在编译后的文件里的类型也就是改对象声明时的类型，而运行时类型是指在程序运行时动态指定的类型也就是该对象定义时的类型。如果编译时类型与运行时类型不一致就会发生运行时动态绑定

比如上面定义的Fruit fruit = new Apple();

fruit的编译时类型是Fruit。

我们可以从编译后的class文件看出

```java
#4 = Class              #243          // com/meiliwu/dragon/model/Apple
#5 = Methodref          #4.#241       // com/meiliwu/dragon/model/Apple."<init>":()V
#6 = Methodref          #244.#245     // com/meiliwu/dragon/model/Fruit.say:()V
```

我们可以从编译后的文件看出。fruit.say方法在编译后指向的是Fruit的say方法。但是运行时类型是Apple。运行时却是调用Apple的say方法。我们从日志可以看出来。

如果我们将代码改成这样

```java
Apple fruit = new Apple();
fruit.say();
```

打印日志如下：

```Java
12-25 11:25:17.803 9709-9709/com.meiliwu.dragon.debug I/Fruit: say:  i am apple
```

我们再看看编译后的文件

```java
4 = Class              #243          // com/meiliwu/dragon/model/Apple
#5 = Methodref          #4.#241       // com/meiliwu/dragon/model/Apple."<init>":()V
#6 = Methodref          #4.#244       // com/meiliwu/dragon/model/Apple.say:()V
```

从代码可以看出编译时类型与运行时类型一致，这是不会发生动态绑定的，这时可以从编译后的class文件得出验证。

JAVA的PECS原则

PECS指“Producer Extends，Consumer Super”

比如：

```java
List<? extends Fruit> fruitList = new ArrayList<>();
fruitList.add(new Apple());
fruitList.add(new Orange());
```

但是编译器却报了编译错误：

按理说fruitList是一个持有类型为Fruit及其子类的泛型列表啊，为什么不能往其中添加Fruit的子类呢？

因为泛型的一大好处就是可以在编译时检查，避免传入不相符的类型可能导致的ClassCastException了。但是声明fruitList的时候没有明确的指定泛型的具体类型，所以编译器无法确认其持有的具体类型，当然也就拒绝了add操作。

fruitList只是规定了泛型的上限，但是并没有确定具体的类型，也无法确定具体的子类型，可以是Apple，Orange还可能是Banana,所以不能把具体的对象添加进去，不然使用的时候可能导致ClassCastException了。但是可以保证从里面取出来的数据都是Fruit及其子类，而且还是Fruit的某一个子类。

我们把代码改成下面这样子就可以添加不同的Fruit对象了。

因为我们规定了fruitList持有的都是Fruit及其父类，可以将Fruit 及其子类都添加进去

```java 
List<? super Fruit> fruitList = new ArrayList<>();
fruitList.add(new Apple());
fruitList.add(new Orange());
```

但是

```Java
fruitList.add(new Object());
```

却不行，why?因为在编译时无法确认具体的fruitList持有的是Fruit的哪一个父类，要想确定就不能用泛型了。所以就无法往里面写入Fruit的父类型对象。

所以我们可以从Java Collctions copy方法的签名可以看出

```Java
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
```

dest集合持有？super T ，可以往里面写入所有T及其子类对象，而src集合持有？ extends T泛型。可以确保的是从里面读取的数据都是T及其子类。所以可以写入dest了。



PS:Java 类的final方法和static不能复写

## Reference ##

1：https://www.cnblogs.com/ygj0930/p/6554103.html

2:  http://www.importnew.com/8966.html