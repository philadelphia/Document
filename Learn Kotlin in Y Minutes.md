# **Learn X in Y minutes** #

	Where X=kotlin 

Kotlin是一种静态类型编程语言，它适用于JVM，Android和浏览器，它百分之百的兼容JAVA。

// 单行注释以`//` 开始

/*
	多行注释
*/

//"package" keyword 与Java里作用一样

Kotlin程序的入口是一个名叫main的函数，该函数的参数为一个包含所有的命令行参数的数组
	
	
	
	package com.learnxinyminutes.kotlin
	
	fun main(args: Array<String>) {
    /*
 
	声明变量可以使用`var`或者`val`
	`val`声明的变量不能被重新赋值，而`var`可以
	/*	
    val fooVal = 10 // we cannot later reassign fooVal to something else
    var fooVar = 10
    fooVar = 20 // fooVar can be reassigned

    /*
	大部分场景下，Kotlin可以决定变量的类型，所有我们不必显式的指定。
	我们可以显式的指定变量的类型，方式如下：
    */
    val foo: Int = 7

    /*
	字符串可以以与Java相同的方式呈现
	转义以一个反斜杠表示
    */
    val fooString = "My String Is Here!"
    val barString = "Printing on a new line?\nNo Problem!"
    val bazString = "Do you want to add a tab?\tNo Problem!"
    println(fooString)
    println(barString)
    println(bazString)

    /*
    A raw string is delimited by a triple quote (""").
    Raw strings can contain newlines and any other characters.
	原始字符从以一个双引号结束，字符串可以包含新的行和任意其他字符。
    */
    val fooRawString = """
    fun helloWorld(val name : String) {
       println("Hello, world!")
    }

    println(fooRawString)

    /*
    Strings can contain template expressions.
    A template expression starts with a dollar sign ($).
	字符串可以包含模板表达式
	一个表达式以$开头
    */
    val fooTemplateString = "$fooString has ${fooString.length} characters"
    println(fooTemplateString)

    /*
    For a variable to hold null it must be explicitly specified as nullable.
    A variable can be specified as nullable by appending a ? to its type.
    We can access a nullable variable by using the ?. operator.
    We can use the ?: operator to specify an alternative value to use
    if a variable is null.
	对于值可以为null的变量，必须显示的声明它为nullable，我们一个在变量的类型后面追加一个`?`来声明一个nullable类型的变量。
	我们可以使用`？.`操作符访问该变量，我们可以使用`？：`操作符来指定一个可用的值，如果该变量的值真的为null.
    */
    var fooNullable: String? = "abc"
    println(fooNullable?.length) // => 3
    println(fooNullable?.length ?: -1) // => 3
    fooNullable = null
    println(fooNullable?.length) // => null
    println(fooNullable?.length ?: -1) // => -1

    /*
    Functions can be declared using the "fun" keyword.
    Function arguments are specified in brackets after the function name.
    Function arguments can optionally have a default value.
    The function return type, if required, is specified after the arguments.

	函数声明以`fun`关键字开头
	函数参数在函数名后的括号里
	函数参数可以指定一个默认值(可选)
	函数的返回值类型指定在参数后面
	如下：
    
	
    fun hello(name: String = "world"): String {
        return "Hello, $name!"
    }
    println(hello("foo")) // => Hello, foo!
    println(hello(name = "bar")) // => Hello, bar!
    println(hello()) // => Hello, world!



    
   
   
	/*一个函数的参数可以以`vararg`标识，表明该参数是一个可变参数*/
    fun varargExample(vararg names: Int) {
        println("Argument has ${names.size} elements")
    }
    varargExample() // => Argument has 0 elements
    varargExample(1) // => Argument has 1 elements
    varargExample(1, 2, 3) // => Argument has 3 elements


    /*当一个函数有只有一个表达式的时候，大括号可以省略，函数体可以跟在`=`后面.*/

    fun odd(x: Int): Boolean = x % 2 == 1
    println(odd(6)) // => false
    println(odd(7)) // => true


    /*如果返回值可以被推断出来，我们没必要指定它*/
    fun even(x: Int) = x % 2 == 0
    println(even(6)) // => true
    println(even(7)) // => false

	//函数可以使用函数作为参数，并且可以返回函数
    
    fun not(f: (Int) -> Boolean): (Int) -> Boolean {
        return {n -> !f.invoke(n)}
    }

	//命名的函数可以被指定为参数，以`：：`操作符标识
    val notOdd = not(::odd)
    val notEven = not(::even)
    // Lambda expressions can be specified as arguments.
	
	//Lambda表达式一个作为参数
    val notZero = not {n -> n == 0}
    /*
    If a lambda has only one parameter
    then its declaration can be omitted (along with the ->).
    The name of the single parameter will be "it".*/
    
	val notPositive = not {it > 0}
    for (i in 0..4) {
        println("${notOdd(i)} ${notEven(i)} ${notZero(i)} ${notPositive(i)}")
    }

    // The "class"关键字被用来声明类
    class ExampleClass(val x: Int) {
        fun memberFunction(y: Int): Int {
            return x + y
        }

        infix fun infixMemberFunction(y: Int): Int {
            return x * y
        }
    }
    /*
	调用构造函数创建一个新的实例，Kotlin没有`new`关键字*/
   
	 val fooExampleClass = ExampleClass(7)
   
	 //可以使用`.`调用函数
	
    println(fooExampleClass.memberFunction(4)) // => 11
  
	  /*
    If a function has been marked with the "infix" keyword then it can be
    called using infix notation.
    */
    println(fooExampleClass infixMemberFunction 4) // => 28

    /*
    Data classes 是一个简洁的方式去创建类持有数据.
    The "hashCode"/"equals" and "toString" 方法自动生成.
    */
    data class DataClassExample (val x: Int, val y: Int, val z: Int)
    val fooData = DataClassExample(1, 2, 4)
    println(fooData) // => DataClassExample(x=1, y=2, z=4)

    // Data classes have a "copy" function.
    val fooCopy = fooData.copy(y = 100)
    println(fooCopy) // => DataClassExample(x=1, y=100, z=4)

----
    // Objects can be destructured into multiple variables.
	对象可以被析构成多个变量
    val (a, b, c) = fooCopy
    println("$a $b $c") // => 1 100 4

    // destructuring in "for" loop
    for ((a, b, c) in listOf(fooData)) {
        println("$a $b $c") // => 1 100 4
    }

    val mapData = mapOf("a" to 1, "b" to 2)
    // Map.Entry is destructurable as well
    for ((key, value) in mapData) {
        println("$key -> $value")
    }

    // The "with" function is similar to the JavaScript "with" statement.
    data class MutableDataClassExample (var x: Int, var y: Int, var z: Int)
    val fooMutableData = MutableDataClassExample(7, 4, 9)
    with (fooMutableData) {
        x -= 2
        y += 2
        z--
    }
    println(fooMutableData) // => MutableDataClassExample(x=5, y=6, z=8)

## Kotlin处理集合 ##
   

	 /*
    We can create a list using the "listOf" function.
    The list will be immutable - elements cannot be added or removed.
    */
    val fooList = listOf("a", "b", "c")
    println(fooList.size) // => 3
    println(fooList.first()) // => a
    println(fooList.last()) // => c
    // Elements of a list can be accessed by their index.
    println(fooList[1]) // => b

    // A mutable list can be created using the "mutableListOf" function.
    val fooMutableList = mutableListOf("a", "b", "c")
    fooMutableList.add("d")
    println(fooMutableList.last()) // => d
    println(fooMutableList.size) // => 4

    // We can create a set using the "setOf" function.
    val fooSet = setOf("a", "b", "c")
    println(fooSet.contains("a")) // => true
    println(fooSet.contains("z")) // => false

    // We can create a map using the "mapOf" function.
    val fooMap = mapOf("a" to 8, "b" to 7, "c" to 9)
    // Map values can be accessed by their key.
    println(fooMap["a"]) // => 8

    /*
    Sequences represent lazily-evaluated collections.
    We can create a sequence using the "generateSequence" function.
    */
    val fooSequence = generateSequence(1, { it + 1 })
    val x = fooSequence.take(10).toList()
    println(x) // => [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

    // An example of using a sequence to generate Fibonacci numbers:
    fun fibonacciSequence(): Sequence<Long> {
        var a = 0L
        var b = 1L

        fun next(): Long {
            val result = a + b
            a = b
            b = result
            return a
        }

        return generateSequence(::next)
    }
    val y = fibonacciSequence().take(10).toList()
    println(y) // => [1, 1, 2, 3, 5, 8, 13, 21, 34, 55]

    // Kotlin provides higher-order functions for working with collections.
    val z = (1..9).map {it * 3}
                  .filter {it < 20}
                  .groupBy {it % 2 == 0}
                  .mapKeys {if (it.key) "even" else "odd"}
    println(z) // => {odd=[3, 9, 15], even=[6, 12, 18]}

## 逻辑控制 ##
    // A "for" loop can be used with anything that provides an iterator.
    for (c in "hello") {
        println(c)
    }

    // "while" 循环的工作机制与其他语言一样
    var ctr = 0
    while (ctr < 5) {
        println(ctr)
        ctr++
    }
    do {
        println(ctr)
        ctr++
    } while (ctr < 10)


	//`if`表达式可以作为返回值，所以三目运算符`？：`在Kotlin中就不在需要了。
    
    val num = 5
    val message = if (num % 2 == 0) "even" else "odd"
    println("$num is $message") // => 5 is odd

    // "when" can be used as an alternative to "if-else if" chains.
    val i = 10
    when {
        i < 7 -> println("first block")
        fooString.startsWith("hello") -> println("second block")
        else -> println("else block")
    }

    // "when" can be used with an argument.
    when (i) {
        0, 21 -> println("0 or 21")
        in 1..20 -> println("in the range 1 to 20")
        else -> println("none of the above")
    }

    // "when" can be used as a function that returns a value.
    var result = when (i) {
        0, 21 -> "0 or 21"
        in 1..20 -> "in the range 1 to 20"
        else -> "none of the above"
    }
    println(result)

##  ##

    /*
	如果一个对象是特殊类型，我们可以使用`is`操作符来检查它，
	如果一个对象通过一个类型检查，它就可以被作为该类型使用而不必显式的转换它
    */
    fun smartCastExample(x: Any) : Boolean {
        if (x is Boolean) {
            // x is automatically cast to Boolean
            return x
        } else if (x is Int) {
            // x is automatically cast to Int
            return x > 0
        } else if (x is String) {
            // x is automatically cast to String
            return x.isNotEmpty()
        } else {
            return false
        }
    }
    println(smartCastExample("Hello, world!")) // => true
    println(smartCastExample("")) // => false
    println(smartCastExample(5)) // => true
    println(smartCastExample(0)) // => false
    println(smartCastExample(true)) // => true

    // Smartcast also works with when block
    fun smartCastWhenExample(x: Any) = when (x) {
        is Boolean -> x
        is Int -> x > 0
        is String -> x.isNotEmpty()
        else -> false
    }

    /*
    
----
	扩展函数是一种给类添加新方法的方式，与`c#`方法类似
    */
    fun String.remove(c: Char): String {
        return this.filter {it != c}
    }
    println("Hello, world!".remove('l')) // => Heo, word!

    println(EnumExample.A) // => A
    println(ObjectExample.hello()) // => hello
	}

	
	// Enum 类与Java中一样
	enum class EnumExample {
	    A, B, C
	}

##  ##

"object" keyword 可以用了创建一个单例对象，我们不能实例化它，但是我们可以使用它的名字引用它到一个唯一的实例
这与Scale单例对象一样.

	object ObjectExample {
	    fun hello(): String {
	        return "hello"
	    }
	}

---
	fun useObject() {
    ObjectExample.hello()
    val someRef: Any = ObjectExample // we use objects name just as is
	}

----------------------------

Further Reading

[  Kotlin tutorials](https://kotlinlang.org/docs/tutorials/)
   
[Try Kotlin in your browser](https://try.kotlinlang.org/)
   
[ A list of Kotlin resources](https://kotlin.link/)
   

   
**原文链接**：**[https://learnxinyminutes.com/docs/kotlin/](https://learnxinyminutes.com/docs/kotlin/)**