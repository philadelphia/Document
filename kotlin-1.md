**# 对象表达式和对象声明 #**

有时候，我们需要创建一个对某个类做了轻微改动的类的对象，而不用为之显式声明新的子类。 Java 用匿名内部类 处理这种情况。 Kotlin 用对象表达式和对象声明对这个概念稍微概括了下。

##  对象表达式 ##

要创建一个继承自某个（或某些）类型的匿名类的对象，我们会这么写：

	window.addMouseListener(object : MouseAdapter() {
    override fun mouseClicked(e: MouseEvent) {
        // ……
    }

    override fun mouseEntered(e: MouseEvent) {
        // ……
    }
	})

如果超类型有一个构造函数，则必须传递适当的构造函数参数给它。 多个超类型可以由跟在冒号后面的逗号分隔的列表指定：

	open class A(x: Int) {
	    public open val y: Int = x
	}
	
	interface B {……}
	
	val ab: A = object : A(1), B {
	    override val y = 15
	}

任何时候，如果我们只需要“一个对象而已”，并不需要特殊超类型，那么我们可以简单地写：

	fun foo() {
	    val adHoc = object {
	        var x: Int = 0
	        var y: Int = 0
	    }
	    print(adHoc.x + adHoc.y)
	}
请注意，匿名对象可以用作只在本地和私有作用域中声明的类型。如果你使用匿名对象作为公有函数的返回类型或者用作公有属性的类型，那么该函数或属性的实际类型会是匿名对象声明的超类型，如果你没有声明任何超类型，就会是 Any。在匿名对象中添加的成员将无法访问。

	class C {
	    // 私有函数，所以其返回类型是匿名对象类型
	    private fun foo() = object {
	        val x: String = "x"
	    }
	
	    // 公有函数，所以其返回类型是 Any
	    fun publicFoo() = object {
	        val x: String = "x"
	    }
	
	    fun bar() {
	        val x1 = foo().x        // 没问题
	        val x2 = publicFoo().x  // 错误：未能解析的引用“x”
	    }
	}
就像 Java 匿名内部类一样，对象表达式中的代码可以访问来自包含它的作用域的变量。 （与 Java 不同的是，这不仅限于 final 变量。）

	fun countClicks(window: JComponent) {
	    var clickCount = 0
	    var enterCount = 0
	
	    window.addMouseListener(object : MouseAdapter() {
	        override fun mouseClicked(e: MouseEvent) {
	            clickCount++
	        }
	
	        override fun mouseEntered(e: MouseEvent) {
	            enterCount++
	        }
	    })
	    // ……
	}
对象声明

单例模式是一种非常有用的模式，而 Kotlin（继 Scala 之后）使单例声明变得很容易：

	object DataProviderManager {
	    fun registerDataProvider(provider: DataProvider) {
	        // ……
	    }
	
	    val allDataProviders: Collection<DataProvider>
	        get() = // ……
	}
这称为对象声明。并且它总是在 object 关键字后跟一个名称。 就像变量声明一样，对象声明不是一个表达式，不能用在赋值语句的右边。

要引用该对象，我们直接使用其名称即可：

	DataProviderManager.registerDataProvider(……)
这些对象可以有超类型：

		object DefaultListener : MouseAdapter() {
	 		   override fun mouseClicked(e: MouseEvent) {
	       // ……
	 		 }
	
	    override fun mouseEntered(e: MouseEvent) {
	        // ……
	    }
	}
注意：对象声明不能在局部作用域（即直接嵌套在函数内部），但是它们可以嵌套到其他对象声明或非内部类中。

伴生对象

类内部的对象声明可以用 companion 关键字标记：

	class MyClass {
	    companion object Factory {
	        fun create(): MyClass = MyClass()
	    }
	}
该伴生对象的成员可通过只使用类名作为限定符来调用：

	val instance = MyClass.create()
可以省略伴生对象的名称，在这种情况下将使用名称 Companion：

	class MyClass {
	    companion object {
	    }
	}

	val x = MyClass.Companion
请注意，即使伴生对象的成员看起来像其他语言的静态成员，在运行时他们仍然是真实对象的实例成员，而且，例如还可以实现接口：

	interface Factory<T> {
	    fun create(): T
	}

	
	class MyClass {
	    companion object : Factory<MyClass> {
	        override fun create(): MyClass = MyClass()
	    }
	}
当然，在 JVM 平台，如果使用 @JvmStatic 注解，你可以将伴生对象的成员生成为真正的静态方法和字段。更详细信息请参见Java 互操作性一节 。

对象表达式和对象声明之间的语义差异

对象表达式和对象声明之间有一个重要的语义差别：

对象表达式是在使用他们的地方立即执行（及初始化）的
对象声明是在第一次被访问到时延迟初始化的
伴生对象的初始化是在相应的类被加载（解析）时，与 Java 静态初始化器的语义相匹配

## 委托 ##

### 类委托 ###

委托模式已经证明是实现继承的一个很好的替代方式， 而 Kotlin 可以零样板代码地原生支持它。 类 Derived 可以继承一个接口 Base，并将其所有共有的方法委托给一个指定的对象：

	interface Base {
	    fun print()
	}
	
	class BaseImpl(val x: Int) : Base {
	    override fun print() { print(x) }
	}
	
	class Derived(b: Base) : Base by b
	
	fun main(args: Array<String>) {
	    val b = BaseImpl(10)
	    Derived(b).print() // 输出 10
	}
Derived 的超类型列表中的 by-子句表示 b 将会在 Derived 中内部存储。 并且编译器将生成转发给 b 的所有 Base 的方法。

请注意，覆盖会以你所期望的方式工作：编译器会使用你的 override 实现取代委托对象中的实现。如果我们为 Derived 添加 
	
	override fun print() {
	 print("abc") 
	}
该程序会输出“abc”而不是“10”。


### 委托属性 ###

有一些常见的属性类型，虽然我们可以在每次需要的时候手动实现它们， 但是如果能够为大家把他们只实现一次并放入一个库会更好。例如包括

延迟属性（lazy properties）: 其值只在首次访问时计算，
可观察属性（observable properties）: 监听器会收到有关此属性变更的通知，
把多个属性储存在一个映射（map）中，而不是每个存在单独的字段中。
为了涵盖这些（以及其他）情况，Kotlin 支持 委托属性:

	class Example {
	    var p: String by Delegate()
	}
语法是：` val/var <属性名>: <类型> by <表达式>`。在 by 后面的表达式是该 委托， 因为属性对应的 get()（和 set()）会被委托给它的 getValue() 和 setValue() 方法。 属性的委托不必实现任何的接口，但是需要提供一个 getValue() 函数（和 setValue()——对于 var 属性）。 例如:

	class Delegate {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        return "$thisRef, thank you for delegating '${property.name}' to me!"
    }
 
    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
        println("$value has been assigned to '${property.name} in $thisRef.'")
    }
	}
当我们从委托到一个 Delegate 实例的 p 读取时，将调用 Delegate 中的 getValue() 函数， 所以它第一个参数是读出 p 的对象、第二个参数保存了对 p 自身的描述 （例如你可以取它的名字)。 例如:

val e = Example()
println(e.p)
输出结果：

Example@33a17727, thank you for delegating ‘p’ to me!
类似地，当我们给 p 赋值时，将调用 setValue() 函数。前两个参数相同，第三个参数保存将要被赋予的值：

e.p = "NEW"
输出结果：

NEW has been assigned to ‘p’ in Example@33a17727.
委托对象的要求规范可以在下文找到。

请注意，自 Kotlin 1.1 起你可以在函数或代码块中声明一个委托属性，因此它不一定是类的成员。 你可以在下文找到其示例。

### 标准委托 ###

Kotlin 标准库为几种有用的委托提供了工厂方法。

延迟属性 Lazy

lazy() 是接受一个 lambda 并返回一个 Lazy <T> 实例的函数，返回的实例可以作为实现延迟属性的委托： 第一次调用 get() 会执行已传递给 lazy() 的 lamda 表达式并记录结果， 后续调用 get() 只是返回记录的结果。
	
	val lazyValue: String by lazy {
	    println("computed!")
	    "Hello"
	}

	fun main(args: Array<String>) {
	    println(lazyValue)
	    println(lazyValue)
	}
这个例子输出：

	computed!
	Hello
	Hello
默认情况下，对于 lazy 属性的求值是同步锁的（synchronized）：该值只在一个线程中计算，并且所有线程会看到相同的值。如果初始化委托的同步锁不是必需的，这样多个线程可以同时执行，那么将 LazyThreadSafetyMode.PUBLICATION 作为参数传递给 lazy() 函数。 而如果你确定初始化将总是发生在单个线程，那么你可以使用 LazyThreadSafetyMode.NONE 模式， 它不会有任何线程安全的保证和相关的开销。

可观察属性 Observable

Delegates.observable() 接受两个参数：初始值和修改时处理程序（handler）。 每当我们给属性赋值时会调用该处理程序（在赋值后执行）。它有三个参数：被赋值的属性、旧值和新值：

	import kotlin.properties.Delegates
	
	class User {
	    var name: String by Delegates.observable("<no name>") {
	        prop, old, new ->
	        println("$old -> $new")
	    }
	}
	
	fun main(args: Array<String>) {
	    val user = User()
	    user.name = "first"
	    user.name = "second"
	}
这个例子输出：

	<no name> -> first
	first -> second
如果你想能够截获一个赋值并“否决”它，就使用 vetoable() 取代 observable()。 在属性被赋新值生效之前会调用传递给 vetoable 的处理程序。

把属性储存在映射中

一个常见的用例是在一个映射（map）里存储属性的值。 这经常出现在像解析 JSON 或者做其他“动态”事情的应用中。 在这种情况下，你可以使用映射实例自身作为委托来实现委托属性。

	class User(val map: Map<String, Any?>) {
	    val name: String by map
	    val age: Int     by map
	}
在这个例子中，构造函数接受一个映射参数：
	
	val user = User(mapOf(
	    "name" to "John Doe",
	    "age"  to 25
	))
委托属性会从这个映射中取值（通过字符串键——属性的名称）：

	println(user.name) // Prints "John Doe"
	println(user.age)  // Prints 25
这也适用于 var 属性，如果把只读的 Map 换成 MutableMap 的话：

	class MutableUser(val map: MutableMap<String, Any?>) {
	    var name: String by map
	    var age: Int     by map
	}
局部委托属性（自 1.1 起）

你可以将局部变量声明为委托属性。 例如，你可以使一个局部变量惰性初始化：

	fun example(computeFoo: () -> Foo) {
	    val memoizedFoo by lazy(computeFoo)
	
	    if (someCondition && memoizedFoo.isValid()) {
	        memoizedFoo.doSomething()
	    }
	}
memoizedFoo 变量只会在第一次访问时计算。 如果 someCondition 失败，那么该变量根本不会计算。

属性委托要求

这里我们总结了委托对象的要求。

对于一个只读属性（即 val 声明的），委托必须提供一个名为 getValue 的函数，该函数接受以下参数：

thisRef —— 必须与 属性所有者 类型（对于扩展属性——指被扩展的类型）相同或者是它的超类型，
property —— 必须是类型 KProperty<*> 或其超类型，
这个函数必须返回与属性相同的类型（或其子类型）。

对于一个可变属性（即 var 声明的），委托必须额外提供一个名为 setValue 的函数，该函数接受以下参数：

thisRef —— 同 getValue()，
property —— 同 getValue()，
new value —— 必须和属性同类型或者是它的超类型。
getValue() 或/和 setValue() 函数可以通过委托类的成员函数提供或者由扩展函数提供。 当你需要委托属性到原本未提供的这些函数的对象时后者会更便利。 两函数都需要用 operator 关键字来进行标记。

委托类可以实现包含所需 operator 方法的 ReadOnlyProperty 或 ReadWriteProperty 接口之一。 这俩接口是在 Kotlin 标准库中声明的：

	interface ReadOnlyProperty<in R, out T> {
	    operator fun getValue(thisRef: R, property: KProperty<*>): T
	}

	interface ReadWriteProperty<in R, T> {
	    operator fun getValue(thisRef: R, property: KProperty<*>): T
	    operator fun setValue(thisRef: R, property: KProperty<*>, value: T)
	}
翻译规则

在每个委托属性的实现的背后，Kotlin 编译器都会生成辅助属性并委托给它。 例如，对于属性 prop，生成隐藏属性 prop$delegate，而访问器的代码只是简单地委托给这个附加属性：

	class C {
	    var prop: Type by MyDelegate()
	}

// 这段是由编译器生成的相应代码：
	class C {
	    private val prop$delegate = MyDelegate()
	    var prop: Type
	        get() = prop$delegate.getValue(this, this::prop)
	        set(value: Type) = prop$delegate.setValue(this, this::prop, value)
	}
Kotlin 编译器在参数中提供了关于 prop 的所有必要信息：第一个参数 this 引用到外部类 C 的实例而 this::prop 是 KProperty 类型的反射对象，该对象描述 prop 自身。

请注意，直接在代码中引用绑定的可调用引用的语法 this::prop 自 Kotlin 1.1 起才可用。

提供委托（自 1.1 起）

通过定义 provideDelegate 操作符，可以扩展创建属性实现所委托对象的逻辑。 如果 by 右侧所使用的对象将 provideDelegate 定义为成员或扩展函数，那么会调用该函数来创建属性委托实例。

provideDelegate 的一个可能的使用场景是在创建属性时（而不仅在其 getter 或 setter 中）检查属性一致性。

例如，如果要在绑定之前检查属性名称，可以这样写：

	class ResourceLoader<T>(id: ResourceID<T>) {
	    operator fun provideDelegate(
	            thisRef: MyUI,
	            prop: KProperty<*>
	    ): ReadOnlyProperty<MyUI, T> {
	        checkProperty(thisRef, prop.name)
	        // 创建委托
	    }
	
	    private fun checkProperty(thisRef: MyUI, name: String) { …… }
	}
	
	fun <T> bindResource(id: ResourceID<T>): ResourceLoader<T> { …… }
	
	class MyUI {
	    val image by bindResource(ResourceID.image_id)
	    val text by bindResource(ResourceID.text_id)
	}
provideDelegate 的参数与 getValue 相同：

thisRef —— 必须与 属性所有者 类型（对于扩展属性——指被扩展的类型）相同或者是它的超类型，
property —— 必须是类型 KProperty<*> 或其超类型。
在创建 MyUI 实例期间，为每个属性调用 provideDelegate 方法，并立即执行必要的验证。

如果没有这种拦截属性与其委托之间的绑定的能力，为了实现相同的功能， 你必须显式传递属性名，这不是很方便：

// 检查属性名称而不使用“provideDelegate”功能
	class MyUI {
	    val image by bindResource(ResourceID.image_id, "image")
	    val text by bindResource(ResourceID.text_id, "text")
	}
	
	fun <T> MyUI.bindResource(
	        id: ResourceID<T>,
	        propertyName: String
	): ReadOnlyProperty<MyUI, T> {
	   checkProperty(this, propertyName)
	   // 创建委托
	}
在生成的代码中，会调用 provideDelegate 方法来初始化辅助的 prop$delegate 属性。 比较对于属性声明 val prop: Type by MyDelegate() 生成的代码与上面（当 provideDelegate 方法不存在时）生成的代码：

	class C {
	    var prop: Type by MyDelegate()
	}

// 这段代码是当“provideDelegate”功能可用时
// 由编译器生成的代码：
	class C {
	    // 调用“provideDelegate”来创建额外的“delegate”属性
	    private val prop$delegate = MyDelegate().provideDelegate(this, this::prop)
	    val prop: Type
	        get() = prop$delegate.getValue(this, this::prop)
	}
请注意，provideDelegate 方ss法只影响辅助属性的创建，并不会影响为 getter 或 setter 生成的代码。