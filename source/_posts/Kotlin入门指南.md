---
title: Kotlin入门指南
date: 2018-04-28 19:07:12
categories:
 - Kotlin
tags:
 - Kotlin
 - 
---


---



## Kotlin的优势
代码简洁高效、强大的when语法，不用写分号结尾，findViewById光荣退休，空指针安全、强大的扩展功能、函数式编程、支持lambda表达式、流式API等等

## Kotlin基本语法
### 基本用法
#### 类型和函数定义
在Kotlin中,常量用`val`声明，变量用`var`声明，关键字在前面，类型在后面以冒号:隔开，也可以省略直接赋值(自动进行类型推断):
```
    var str: String = "hello"  //字符串
    var a: Int = 10  //整形
    var array: Array<Int> = arrayOf(1, 2, 3)  //数组
    var str2: String? = null  //可空字符串变量
```
<!-- more -->

定义一个函数接受两个 int 型参数，返回值为 int ：
```
    fun sum(a: Int, b: Int): Int {
        return a+b
    }
```
该函数只有一个表达式函数体以及一个可以推断类型的返回值，因此可以简写为：
```
    fun sum(a: Int, b: Int) = a + b
```


无返回值的函数：
```
    fun printSum(a: Int, b: Int): Unit {   //一般，Unit可以省略不写
        println("sum of $a and $b is ${a + b}")
    }
    //也可简写为：
    fun printSum(a: Int, b: Int) = println("sum of $a and $b is ${a + b}")
```
函数参数可以设置默认值,当参数被忽略时会使用默认值。这样相比其他语言可以减少重载：
```
    fun sum(a: Int, b: Int = 1) = a + b
```
另外Kotlin还支持扩展函数和中缀表达式，下面是简单的例子：
```
//扩展函数
fun String.isLetter(): Boolean {
    return matches(Regex("^[a-z|A-Z]$"))
}

//中缀表达式只能是成员函数或者扩展函数，而且函数只有一个参数
infix fun String.isEqual(that: String): Boolean{
    return this == that
}

fun main(args: Array) {
    var s = "a".isLetter()
    var a = "aa" isEqual "bb"
}
```
String对象中本没有判断是否是字母的方法，在Java中我们一般会定义一些Utils方法，而在Kotlin中可以定义类的扩展函数。
第二个例子是给String类定义了一个扩展函数，并且该拓展函数以中缀表达式表示，给予了开发者定义类似于关键字的权利。

再比如，我们一般这样创建一个map对象：
```
val kv = mapOf("a" to 1, "b" to 2)
```
这里的to就是一个中缀表达式，定义如下：
```
public infix fun<A, B> A.to(that: B): Pair<A, B> = Pair(this, that)
```
Pair就是Map中存的对象，所以你也可以这样创建:
```
val kv = mapOf(Pair("a", 1), Pair("b", 2))
```

### 流程控制语句
流程控制语句是编程语言的核心之一。跟java类似，Kotlin有以下的语句。

 - 分支语句 if、when
 - 循环语句 for、while
 - 跳转语句 return、break、continue、throw
 
#### if表达式
在Kotlin中，if是一个表达式，即它会返回一个值。if作为代码块时，最后一行作为返回值。
```
fun max(x: Int, y: Int): Int {
    return if (x > y) {
        println("max is $x")
        x
    }else{
        println("max is $y")
        y
    }
}
```
注意：**Kotlin中没有java的中的2>1?2:1这样的三元表达式。**

#### when表达式
Kotlin中没有java中的switch-case表达式，when表达式就是用来代替switch-case的。when会对所有的分支进行检查直到有一个条件满足。但相比switch而言，when语句要更加的强大，灵活：
```
fun cases(obj: Any) {
    when (obj) {
        1 -> print("第一项")
        "hello" -> print("这个是字符串hello")
        is Long -> print("这是一个Long类型数据")
        !is String -> print("这不是String类型的数据")
        else -> print("else类似于Java中的default")
    }
}
```
如果我们有很多分支需要用相同的方式处理，则可以把多个分支条件放在一起，用逗号分隔，
也可以用任意表达式（而不只是常量）作为分支条件，也可以检测一个值在 in 或者不在 !in 一个区间或者集合中：
```
when (x) {
        1 -> print("x == 1")
        2 -> print("x == 2")
        3, 4 -> print("x == 3 or x == 4")
        in 5..9 -> print("x in [5..9]")
        is Long -> print("x is Long")
        !in 10..20 -> print("x is outside the range")
        parseInt(s) -> print("s encodes x")
        else -> { 
            print("x is funny")
        }
    }
```
其他for,while,break,continue等，跟Java中用法基本一样。

### NPE空指针的处理
Kotlin的空指针处理相比于java有着极大的提高，可以说是不用担心出现NullPointerException的错误，kotlin对于对象为null的情况有严格的界定，编码的阶段就需要用代码表明引用是否可以为null，为null的情况需要强制性的判断处理。
#### 可选型定义
##### 非空类型
先说可选型的定义，当我们在Kotlin中定义一个变量时，默认就是非空类型的，当你将一个非空类型置空的时候，编译器会告诉你这不可行。例如：
```
var a: String=null //编译器直接报错 Null can not be value of a non null type string
```
**这里注意：Kotlin的成员变量(全局变量)必须要初始化，甚至是基本数据类型都要手动给一个初始值，局部变量可以不用初始化，上面的例子是成员变量的声明。**

编译器直接表示a是一个non null type, 你不可以直接赋一个null值。对于我们java原住民来说声明变量时如果不去赋值，编译器会默认赋null(除去基本数据类型)，但在Kotlin这是不允许的。

##### 可选型（可空类型）
在定义可选型的时候，我们只要在非空类型的后面添加一个 ? 就可以了。
```
var b: String? = null  //ok
```
在使用可选型变量的时候，这个变量就有可能为空，所以在使用前我们应该对其进行空判断（在 Java 中我们经常这样做），这样往往带来带来大量非业务相关的工作，这些空判断代码本身没有什么实际意义，并且让代码的可读性和简洁性带来了巨大的挑战。
Kotlin 为了解决这个问题，它并不允许我们直接使用一个可选型的变量去调用方法或者属性。例如：
```
var b: String? = "abc"
val l = b.length // compilation error
```
你可以和 Java 中一样，在使用变量之前先进行空判断，然后再去调用。如果使用这种方法，那么空判断是必须的。
```
val length = if (b != null) b.length else -1
```
**注意： 如果你定义的变量是全局变量，即使你做了空判断，依然不能使用变量去调用方法或者属性。**
Kotlin 为可选型提供了一个安全调用操作符 ?.，使用该操作符可以方便调用可选型的方法或者属性。
```
var length = b?.length  //length类型是可选型Int?
```
Kotlin还提供了一个强转的操作符!!，这个操作符能够强行调用变量的方法或者属性，而不管这个变量是否为空，如果这个时候该变量为空时，那么就会发生 NPE。所以如果不想继续陷入NPE 的困境无法自拔，请尽量不要选用该操作符。
```
var length = b!!.length  //可能会报NPE错误（Caused by: kotlin.KotlinNullPointerException）
```
##### Elvis 操作符?:
回到`?.`的调用上来，这个调用方式存在一个让人不安的处理，就是在变量为null的情况下，会直接返回null，这样空指针的隐患还在。
```
var b: String? = "abc"
var length : Int = b?.length //报错类型不匹配 Int? 和 Int
```
修正的话,需要通过if判断来进行判空处理
```
val length = if (b != null) b?.length else -1
```
这种写法可以简化成Elvis 操作符`?:`，例如
```
val length = b?.length ?: -1
```
`b?.length ?: -1` 和 `if (b != null) b.length else -1` 完全等价的。
其实你还可以在`?:` 后面添加任何表达式，比如你可以在后面用`return`和`throw`（在 Kotlin 中它们都是表达式）。

## Kotlin函数式编程
下面是维基百科上对于函数式编程的定义：
>函数式编程（英语：functional programming）或称函数程序设计，又称泛函编程，是一种编程范型，它将电脑运算视为数学上的函数计算，并且避免使用程序状态以及易变对象。函数编程语言最重要的基础是λ演算（lambda calculus）。而且λ演算的函数可以接受函数当作输入（引数）和输出（传出值）。

下面是关于高阶函数的定义：

>在数学和计算机科学中，高阶函数是至少满足下列一个条件的函数：接受一个或多个函数作为输入，输出一个函数

```
f(x) = x^2
g(x) = x + 1
g(f(x))就是一个高阶函数
```

不难推断出函数式编程最重要的基础是高阶函数。也就是支持函数可以接受函数当作输入（引数）和输出（传出值）。

函数式编程的精髓在于函数本身。在函数式编程中函数是第一等公民，与其他数据类型一样，处于平等地位，可以赋值给其他变量，也可以作为参数，传入另一个函数，或者作为别的函数的返回值。
### 从Lambda表达式说起
Lambda 表达式俗称匿名函数，熟悉Java的大家应该也明白这是个什么概念。Kotlin 的 Lambda表达式更“纯粹”一点， 因为它是真正把Lambda抽象为了一种类型，而 Java 8 的 Lambda 只是单方法匿名接口实现的语法糖罢了。
```
        view.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Toast.makeText(v.getContext(), "Click", Toast.LENGTH_SHORT).show();
            }
        });
```
这可以转换为Kotlin代码（使用Anko库函数toast）：
```
view.setOnClickListener(object : View.onClickListener {
    override fun onClick(v: View) {
        toast("Click")
    }
}
```
Lambda表达式由箭头左侧函数的参数（小括号里面的内容）定义的，将值返回给箭头右侧。在这里，将得到的View返回给Unit，这样可以对上述代码稍做简化：
```
view.setOnClickListener({ view -> toast("Click")})
```
在定义函数时，必须在箭头左边使用中括号并指定参数值，而函数执行代码在右边。
如果左边没有使用参数，甚至可以省去左边部分：
```
view.setOnClickListener({ toast("Click") })
```
如果被执行的函数是当前函数的最后一个参数的话，也可以将这个作为参数的函数放到圆括号外面：
```
view.setOnClickListener() { toast("Click") }
```
最后，如果函数是唯一的参数，还可以去掉原来的小括号：
```
view.setOnClickListener { toast("Click") }
```
比起起初的Java代码，目前的代码量小于原来的五分之一，且更容易理解。令人印象深刻。Anko给出一个（本质上说是函数名的）简化版本，由之前展示过的扩展函数组成，该函数也由上述形式实现：
```
view.onClick { toast("Click") }
```
以上是Java中的接口映射到Kotlin中Lambda表达式实例。
### Kotlin中Lambda表达式

下面详细介绍Kotlin中Lambda表达式。
```
var add = {x: Int, y: Int -> x + y}
```
我们可以这样使用：
```
fun main(args: Array) {
val add = {x: Int, y: Int -> x + y}
add.invoke(1, 2)
//或者简写为
add(1, 2)
}
```
它可以像函数一样使用()调用，在kotlin中操作符是可以重载的，()操作符对应的就是类的重载函数invoke()。
还可以想下面这样定义一个变量：
```
val numFun: (a: Int, b: Int) -> Int
```
它不是一个普通的变量，它必须指向一个函数，并且函数签名必须一致：
```
fun main(args: Array) {
    val sumLambda = {a: Int, b: Int -> a + b}
    var numFun: (a: Int, b: Int) -> Int
    numFun = {a: Int, b: Int -> a + b}
    numFun = sumLambda
    numFun = ::sum
    numFun(1,2)
}

fun sum(a: Int, b: Int): Int {
    return a + b
}
```
可以看到这个变量可以等于一个lambda表达式，也可以等于另一个lambda表达式变量，还可以等于一个普通函数，但是在函数名前需要加上(::)来获取函数引用，有点类似于C++中的函数指针。

我们还可以将一个函数传递给另一个函数，比如：
```
fun <T> doMap(list: List<T>, function: (it: T) -> Unit) {
        for (item in list) {
            function(item)
        }
    }
```
第一个参数是一个List，第二个参数是一个函数，目的就是将List中的每一个元素都执行一次第二个函数。使用方法如下：
```
val strList = listOf("a" ,"b", "c", "d")
doMap(strList, {item -> println("item: $item, ") })
```
第二个参数直接传进去了一个lambda表达式，当然也可以传一个函数引用：
```
doMap(strList, ::printLetter)
fun printLetter(item: String) {
    println("item: $item, ")
}
```
效果和上面的代码一样。

另外Kotlin还支持局部函数和函数作为返回值，看下面的代码：
```
fun main(args: Array) {
        val addResult = lateAdd(2, 4)
        addResult()
    }
    //局部函数，函数引用
    fun lateAdd(a: Int, b: Int): ()->Int {
        fun add(): Int {
            return a + b
        }
        return ::add
    }
```
在lateAdd内部定义了一个局部函数，最后返回了该局部函数的引用，对结果使用()操作符拿到最终的结果，达到延迟计算的目的。

基于以上函数式编程的特性，Kotlin可以像RxJava一样很方便的进行响应式编程，比如：
计算二维数组每一个子列表的乘积，再求和，可以这样写：
```
            val list = listOf(
                    listOf(1, 2, 3),
                    listOf(4, 5, 6),
                    listOf(7, 8, 9)
            )
            list.map {it: List<Int> -> it.fold(1){ a, b -> a * b } }  
                .fold(0){ a, b -> a + b }
                .log()

```


### Kotlin函数式编程中常用函数 forEach,filter,map,reduce(fold)
#### **forEach**
遍历函数，循环遍历所有元素，元素是it，可对每个元素进行相关操作；
假设我们现在需要打印列表中每个的名字：
```
val nameList = arrayOf("Jim","Tom", "Marry","Lin")
nameList.forEach { println(it) }
```

#### **filter**
过滤函数将用户给定的布尔逻辑作用于集合，返回由原集合中符合条件的元素组合的一个子集。假设一个逻辑，将数组中是2的倍数的数筛选出来，使用Kotlin和Java的实现做一个简单的对比。
```
int[] all = {1, 2, 3, 4, 5, 6, 7, 8, 9};
List<Integer> filters = new ArrayList<>();
for (int a : all) {
    if (a % 2 == 0) {
        filters.add(a);
    }
}
```
Kotlin代码的实现：
```
val all = arrayOf(1, 2, 3, 4, 5, 6, 7, 8, 9)
val filters = all.filter { it % 2 == 0 }
```
Kotlin 还提供一系列类似的过滤函数：

 - filterIndexed, 同 filter，不过在逻辑判断的方法块中可以拿到当前item的index
 - filterNot，与filter相反，只返回不符合条件的元素组合
 - 针对 Map 类型数据集合，提供了 filterKeys 和 filterValues 方法，方便只做key或者value的判断

#### **map**
映射函数也是一个高阶函数，将一个集合经过一个传入的变换函数映射成另外一种集合。
假设我们现在需要将一系列的名字的字符串长度保存到另一个数组:
使用Java实现过程如下：
```
String[] names = {"James", "Tommy", "Jim", "Andy"};
int[] namesLength = new int[names.length];
for (int i = 0; i < names.length ; i ++) {
    namesLength[i] = names[i].length();
}
```
使用Kotlin实现如下：
```
val names = arrayOf("James", "Tommy", "Jim", "Andy");
val namesLength = names.map { it.length }
```
同 filter 相似，Kotlin 也提供的 mapIndexed 的类似方法方便使用，针对 Map 类型的集合也有 mapKeys 和 mapValues 的封装。
#### **reduce**
归纳函数将一个数据集合的所有元素通过传入的操作函数实现数据集合的积累叠加。同fold一样，不过fold可以带初始值。
假设我们现在需要计算一系列的名字的总字符串长度，使用Java实现如下：
```
String[] names = {"James", "Tommy", "Jim", "Andy"};
int totalLength = 0;
for (int i = 0; i < names.length ; i ++) {
    totalLength =+ names[i].length();
}
```
使用Kotlin实现如下：
```
val names = arrayOf("James", "Tommy", "Jim", "Andy");
val namesLength = names.map { it.length }.reduce { r, s -> r + s }
```

## 关于Anko
Jetbrains给Android带来的不仅是Kotlin，还有Anko。从Anko的官方说明来看这是一个雄心勃勃的要代替XML写Layout的新的开发方式。Anko最重要的一点是引入了DSL（Domain Specific Language 领域相关语言）的方式开发Android界面布局。当然，本质是代码实现布局。不过使用Anko完全不用经历Java纯代码写Android的痛苦。

然而，这个不是我们能在这个库中得到的唯一一个功能。Anko包含了很多的非常有帮助的函数和属性来避免让你写很多的模版代码。多看看Anko源码的实现方式对学习Kotlin语言是非常有帮助的。

Anko提供了非常简单的DSL来处理异步任务，它能够满足大部分的需求。它提供了一个基本的doAsync函数用于在子线程执行代码，可以选择通过调用uiThread的方式回到主线程。在子线程中执行请求如下这么简单：
```
        doAsync {
            Thread.sleep(3000)   //耗时操作
            uiThread {
                toast("background task finish")
            }
        }
```
UIThread有一个很不错的一点就是可以依赖于调用者。如果它是被一个Activity调用的，那么如果activity.isFinishing()返回true，则uiThread不会执行，这样就不会在Activity销毁的时候遇到崩溃的情况了。

## Kotlin中的类和对象
### Kotlin 的类特性

Kotlin中的类与接口和Java中的有些区别：

 - Kotlin中接口可以包含属性申明
 - Kotlin的类申明，默认是final和public的
 - Kotlin的嵌套类并不是默认在内部的。它们不包含外部类的隐式引用
 - Kotlin的构造函数，分为主构造函数和次构造函数
 - Kotlin中可以使用data关键字来申明一个数据类
 - Kotlin中可以使用object关键字来表示单例对象、伴生对象等

Kotlin类的成员可以包含：

 - 构造函数和初始化块
 - 属性
 - 函数
 - 嵌套类和内部类
 - 对象声明
 
### 构造函数
在Kotlin中可以有一个主构造函数，一个或者多个次构造函数。
#### 主构造函数
主构造函数直接跟在类名后面，如下：
```
open class Person constructor(var name: String, var age: Int) : Any() {
    ...
}
```
主构造函数中的属性可以是可变的（var）也可以是不变的（val）。如果主构造函数没有任何注解或者可见性修饰符，可以省略constructor关键字（属性默认是val），而且Koltin中的类默认就是继承Any的，也可以省略。所以可以简化成如下：
```
open class Person(name: String, age: Int) {
    ...
}
```
主构造函数不能包括任何代码。初始化代码可以放到以init关键字作为前缀的初始化块中：
```
open class Person constructor(var name: String, var age: Int){
    init {
        println("Student(name = $name, age = $age) created")
    }
}
```
主构造函数的参数可以在初始化块中使用，也可以在类体内申明的属性初始化器中使用。
#### 次构造函数
我们也可以在类体中使用constructor申明次构造函数，次构造函数的参数不能使用val或者var申明。
```
class Student public constructor(name: String, age: Int) : Person(name, age) {
    var grade: Int = 1

    init {
        println("Student(name = $name, age = $age) created")
    }

    constructor(name: String, age: Int, grade: Int) : this(name, age){
        this.grade = grade
    }
}
```
如果一个类有主构造函数，那么每个次构造函数都需要委托给主构造函数，委托到同一个类的另一个构造函数可以使用this关键字，如上面这个例子this(name, age)

如果一个非抽象类没有申明任何构造函数（包括主或者次），它会生成一个不带参数的主构造函数。构造函数的可见性是public。

### 抽象类
下面就是一个抽象类，类需要用abstract修饰，其中也有抽象方法，跟Java有区别的是Kotlin的抽象类可以包含抽象属性：
```
abstract class Person(var name: String, var age: Int){

    abstract var addr: String
    abstract val weight: Float

    abstract fun doEat()
    abstract fun doWalk()

    fun doSwim() {
        println("I am Swimming ... ")
    }

    open fun doSleep() {
        println("I am Sleeping ... ")
    }
}
```
在上面这个抽象类中，有doEat和doWalk抽象函数，同时还有具体的实现函数doSwim，在Kotlin中如果类和方法没有修饰符的化，默认就是final的。这个跟Java是不一样的。所以doSwim其实是final的，也就是说Person的子类不能覆盖这个方法。如果一个类或者类的方法想要设计成被覆盖（override）的，那么就需要加上open修饰符。下面是一个Person的子类：
```
class Teacher(name: String, age: Int) : Person(name, age) {

    override var addr:String = "Guangzhou"
    override val weight: Float = 100.0F

    override fun doEat() {
        println("Teacher is Eating ... ")
    }

    override fun doWalk() {
        println("Teacher is Walking ... ")
    }

    override fun doSleep() {
        super.doSleep()
        println("Teacher is Sleeping ... ")
    }
    
//    编译错误，doSwim函数默认是final的，需要加上open修饰符才能重写
//    override fun doSwim() {
//        println("Teacher is Swimming ... ")
//    }

}
```
如果子类覆盖了父类的方法或者属性，需要使用override关键字修饰。如果子类没有实现父类的抽象方法，或者没有给抽象属性赋值，则必须把子类也定义成抽象类。

抽象函数的特征有以下几点：

 - 抽象函数、抽象属性必须使用abstract关键字修饰
 - 抽象函数或者抽象类不用手动添加open关键字，默认就是open类型
 - 抽象函数没有具体的实现，抽象属性不用赋值
 - 含有抽象函数或者抽象属性的类，必须要使用abstract关键字修饰。抽象类可以有具体实现的函数，这样的函数默认是final的（不能被覆盖）。如果想要被覆盖，需要手工加上open关键字

### 接口
和Java类似，Kotlin使用interface作为接口的关键词，Kotlin 的接口与 Java 8 的接口类似。与抽象类相比，他们都可以包含抽象的方法以及方法的实现:
```
interface ProjectService {
    val name: String
    val owner: String
    fun save(project: Project)
    fun print() {
        println("I am project")
    }
}
```
接口是没有构造函数的，和继承一样，我们也是使用冒号:来实现一个接口，如果要实现多个接口，使用逗号,分开。

### 继承
在Kotlin中，所有的类都默认继承Any这个类，Any并不是跟Java的java.lang.Object一样，因为它只有equals()，hashCode()和toString()这三个方法。
除了抽象类和接口默认是可被继承外，其他类默认是不可以被继承的（相当于默认都带有final修饰符）。而类中的方法也是默认不可以被继承的。

 - 如果你要继承一个类，你需要使用open关键字修饰这个类
 - 如果你要重写一个类的某个方法，这个方法也需要使用open关键字修饰
 
```
open class Base(type: String){
    open fun canBeOverride() {
        println("I can be override.")
    }

    fun cannotBeOverride() {
        println("I can't be override.")
    }
}

class SubClass(type: String) :Base(type){
    override fun canBeOverride() {
        super.canBeOverride()
        println("Override!!!")
    }

//    override fun cannotBeOverride() {  编译错误。
//        super.cannotBeOverride()
//        println("Override!!!")
//    }
}
```
如果Base有构造函数，那么子类的主构造函数必须继承。

### 单例模式
Kotlin中没有静态属性和方法，但是也提供了实现类似单例的功能，使用object关键字声明一个object对象。
```
object StringUtils{
    val separator: String = """\"""
    fun isDigit(value: String): Boolean{
        for (c in value) {
            if (!c.isDigit()) {
                return false
            }
        }
        return true
    }
}

fun main(args: Array<String>) {
    println("C:${StringUtils.separator}Users${StringUtils.separator}Denny") //打印c:\Users\Denny
    println(StringUtils.isDigit("12321231231"))  //打印true
}
```
我们反编译后可以知道StringUtils转成了Java代码：
```
public final class StringUtils {
   @NotNull
   private static final String separator = "\\";
   public static final StringUtils INSTANCE;

   @NotNull
   public final String getSeparator() {
      return separator;
   }

   public final boolean isDigit(@NotNull String value) {
      Intrinsics.checkParameterIsNotNull(value, "value");
      String var4 = value;
      int var5 = value.length();

      for(int var3 = 0; var3 < var5; ++var3) {
         char c = var4.charAt(var3);
         if (!Character.isDigit(c)) {
            return false;
         }
      }

      return true;
   }

   static {
      StringUtils var0 = new StringUtils();
      INSTANCE = var0;
      separator = "\\";
   }
}
```
object对象只能通过对象名字来访问，不能使用构造函数。
我们在Java中通常会写一些Utils类，这样的类我们在Kotlin中就可以直接使用object对象。

### 伴生对象(companion object)
Kotlin中还提供了伴生对象 ，跟java中的static关键字有些相似，用companion object关键字声明:
```
class DataProcessor {
    fun process() {
        println("Process Data")
    }
    
    companion object StringUtils {    //StringUtils可以省略
        val TAG = "DataProcessor"
        fun isEmpty(s: String): Boolean {
            return s.isEmpty()
        }
    }

}
```
一个类只能有一个伴生对象，伴生对象的初始化是在相应的类被加载解析时，即使伴生对象的成员看起来像其他语言的静态成员，在运行时他们仍然是真实的对象的实例成员。

另外，如果想使用Java中的静态成员和静态方法的话，我们可以用：

 - **@JvmField**注解：生成与该属性相同的静态字段
 - **@JvmStatic**注解：在单例对象和伴生对象中生成对应的静态方法

 ### 嵌套类（Nested Class）
```
class NestedClassesDemo {
    class Outer {
        private val zero: Int = 0
        val one: Int = 1

        class Nested {
            fun getTwo() = 2
            class Nested1 {
                val three = 3
                fun getFour() = 4
            }
        }
    }
}
```
我们可以这样来访问内部类：
```
val one = NestedClassesDemo.Outer().one
val two = NestedClassesDemo.Outer.Nested().getTwo()
val three = NestedClassesDemo.Outer.Nested.Nested1().thre
```
但是普通的嵌套类，并不会持有外部类的引用，如果要持有外部类的引用，那么我们需要把嵌套类标记为内部类，使用inner关键字即可：
```
class NestedClassesDemo {
class Outer {
        private val zero: Int = 0
        val one: Int = 1

        inner class Inner {
            fun accessOuter() = {
                println(zero) // works
                println(one) // works
            }
        }
}
```

### 匿名内部类（Annonymous Inner Class）
匿名内部类，就是没有名字的内部类。既然是内部类，那么它自然也是可以访问外部类的变量的。
我们使用对象表达式创建一个匿名内部类实例：
```
        button.setOnClickListener(object : View.OnClickListener{
            override fun onClick(v: View) {
                println("Clicked")
            }
        })
```
如果对象实例是一个函数接口（Java中只有一个抽象方法的接口），可以使用lambda表达式:
```
        button.setOnClickListener({view -> println("Clicked")})
```
### 对象表达式（Object expressions）
如果父类型有构造函数，则必须将构造函数的参数赋值；多个父类通过“,”分割：
```
open class A(x: Int) {  
    public open val y: Int = x  
}  
  
interface B {...}  
  
val ab: A = object : A(1), B {  
    override val y = 15  
}  
```
有时，只需要一个对象表达式，不想继承任何的父类型，实现如下：
```
val adHoc = object {  
  var x: Int = 0  
  var y: Int = 0  
}  
print(adHoc.x + adHoc.y)  
```
### 对象表达式和对象声明的区别

 - 对象表达式(Object expressions)，在它们使用的地方，是立即（immediately）执行（或初始化）
 - 对象声明(Object declarations)，会延迟（lazily）初始化；但第一次访问该对象时才执行
 - 伴生对象（Companion Objects），当外部类被加载时初始化，跟Java静态代码框初始化相似
## 总结
以上简单介绍了一下Kotlin的基本语法，函数式编程和Kotlin类和对象的入门知识。由于这只是一篇入门的文章，对于Kolin中更多的知识点，比如泛型，协程，与Java的互调等更深层的内容并没有介绍。各位读者如果想要深入学习Kotlin编程，可以参考下面的Kotlin学习进行学习。

---

> * 路漫漫其修远兮，吾将上下而求索。
> * 怕什么真理无穷，进一寸有一寸的欢喜。

---

 
 
## Kotlin学习资料
1. [Kotlin基础教程][1]
2. [《Kotlin for android developers》中文版翻译][2]
3. [Kotlin的类和对象][3]
4. [Kotlin面向对象笔记][4]
5. [Object Expressions and Declarations][5]


  [1]: https://alleniverson.gitbooks.io/kotlin-tutorials/content/
  [2]: https://alleniverson.gitbooks.io/kotlin-for-android-developers/content/
  [3]: https://www.jianshu.com/p/9c9b0a8ab4cf?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation
  [4]: https://www.jianshu.com/p/037dd04b3186
  [5]: http://kotlinlang.org/docs/reference/object-declarations.html
