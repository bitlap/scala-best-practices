## 2. Language Rules

<img src=".././assets/scala-logo-256.png"  align="right" width="128" height="128" />

### 2.1. MUST NOT use "return"

> 不要使用 `return`

Java 的 `return` 语句会产生副作用，即释放堆栈并将此值交给调用者。在一种强调全副作用的编程语言中，这样做是合理的。然而，Scala 是一种面向表达式的语言，其重点在于控制/限制副作用，因此 `return` 语句并不习惯。

更糟糕的是，`return` 的行为可能并不像你想象的那样。例如，在 Play 控制器中，尝试执行以下操作：
```scala
def action = Action { request =>
  if (someInvalidationOf(request))
    return BadRequest("bad")

 Ok("all ok")
}
```

在 Scala 中，嵌套匿名函数内的 `return` 语句是通过抛出和捕获 `NonLocalReturnException` 来实现的。[Scala 语言规范](https://scala-lang.org/files/archive/spec/2.13/spec.pdf) 第 6.20 节是这么说的。

此外，`return` 是反结构编程，因为函数可以有多个退出点，如果你需要 `return`，比如在那些有大量 `if/else` 分支的巨大方法中，`return` 的存在是一个明确的信号，表明代码很糟糕，未来很容易引起BUG，因此急需重构。

### 2.2. SHOULD use immutable data-structures

> 应该使用不可变数据结构

不可变的数据结构是可以比较和推理的事实。可变的东西是容易出错的桶（掉坑）。不应该使用可变数据结构，除非你能保护它，但是能保护可变数据结构的地方又是很少的。

让我们举例说明：

```scala
trait Producer[T] {
 def fetchList: List[T]
}

// 消费端
someProducer.fetchList
```

问题 - 如果上面返回的 `List` 是可变的，这对 `Producer` 接口意味着什么？

这里存在一些问题：
1. 如果这个 `List` 是在消费者之外的另一个线程上生成的，则可能会同时出现可见性和原子性问题 —— 你无法知道是否会发生这种情况，除非你查看生产者的实现。
2. 即使这个列表实际上是不可变的（即仍然可变，但在向消费者发出信号后就不再被修改），你也不知道它是否会向其他可能自行修改它的消费者发出信号，因此你无法推理能用它做什么。
3. 即使描述对此列表的访问必须同步，问题是 —— 你打算同步哪个锁？你能确定锁的正确顺序吗？锁是不可组合的。

所以，这就是问题所在 —— 暴露可变数据结构的公共应用程序接口是一种可恶的行为，它导致的问题可能比手动内存管理更糟糕。

### 2.3. SHOULD NOT update a `var` using loops or conditions

> 不应该使用循环和条件更新 `var`

这是大多数 Java 开发人员在使用 Scala 时都会犯的错误。例子：
```scala
var sum = 0
for (elem <- elements) {
  sum += elem.value
}
```

避免这样做，而是使用函数，如 `foldLeft`：
```scala
val sum = elements.foldLeft(0)((acc, e) => acc + e.value)
```

或者更好的办法是，了解标准库，并始终偏向于使用内置函数 —— 表现力越强，BUG就越少：
```scala
val sum = elements.map(_.value).sum
```

本着同样的精神，你不应该使用条件更新部分结果。例如：
```scala
def compute(x) = {
  var result = resultFrom(x)

  if(needToAddTwo) {
    result += 2
  }
  else {
    result += 1
  }

  result
}
```

首选表达式和不变性。代码的可读性将会更高，不易出错，因为它使分支更明确，这是件好事：
```scala
def computeResult(x) = {
  val r = resultFrom(x)
  if (needToAddTwo)
    r + 2
  else
    r + 1
}
```

而你知道，只要分支变得过于复杂，就像关于 `return` 的讨论中所说的那样，这就表明代码有*异味*，需要重构，这是一件好事。

### 2.4. SHOULD NOT define useless traits

> 不应该定义无用的特质

Java 的最佳实践是 "根据接口编程，而不是根据实现编程"。一般来说，这条规则是健康的，但它指的是隐藏实现细节（尤其是修改状态的细节）的一般工程需求（封装），而不是指经常泄露实现细节的接口声明。

定义特质对代码读者来说也是一种负担，因为这意味着需要多态性。举例说明：
```scala
trait PersonLike {
  def name: String
  def age: Int
}

case class Person(name: String, age: Int)
  extends PersonLike
```

阅读这段代码的读者可能会得出这样的结论：在某些情况下，覆盖 `PersonLike` 是可取的。这与事实相去甚远 —— `Person` 在其样例类中被完美地描述为一种无行为的数据结构。换句话说，它描述了数据的形状，如果你因为某种未知的原因需要覆盖这种形状，那么这个特质的定义就很糟糕，因为它强加了数据的形状，而这是你唯一可以覆盖的东西。如果你需要多态性，需求发生变化之后，你可以随时提出特质。

如果你认为你可能需要覆盖这个源（比如在第一次访问时从数据库中获取人名），请不要这么做！

请注意，我不是在谈论代数数据结构（即表示封闭选择集的密封特质 —— 例如 `Option`）。

即使在你认为问题已经很清楚的情况下，也未必如此。让我们举个例子：
```scala
trait DBService {
  def getAssets: Future[Seq[(AssetConfig, AssetPersistedState)]]

  def persistFlexValue(flex: FlexValue): Future[Unit]
}
```

这段代码来自真实场景 —— 我们有一个 `DBService`，它可以处理查询或数据库中的持久化。这两种方法实际上是不相关的，所以如果你只需要获取资产，为什么要依赖那些需要数据库交互的组件中你不需要的东西呢？

最近，我的代码看起来像这样：

```scala
final class AssetsObservable
    (f: => Future[Seq[(AssetConfig, AssetPersistedState)]])
  extends Observable[AssetConfigEvent] {

  // ...
}

object AssetsObservable {
  // constructor
  def apply(db: DBService) = new AssetsObservable(db.getAssets)
}
```

瞧，我不需要模拟整个`DBService`来测试上述功能。

### 2.5. MUST NOT use "var" inside a case class

> 不要在样例类中使用 `var`

样例类是定义类的语法糖，其中所有构造函数参数都是公共的和不可变的，因此是值标识的一部分，具有结构相等性、相应的 `hashCode` 实现以及编译器提供的自动生成的 `apply/unapply` 函数。

通过这样做：

```scala
case class Sample(str: String, var number: Int)
```

你刚刚破坏了它的相等性和 `hashCode` 操作。现在尝试将其用作map中的键。

一般来说，结构相等只适用于不可变的事物，因为相等操作必须是稳定的（不会随对象的历史而改变）。样例类适用于严格不可变的事物。如果需要更改对象，就不要使用样例类。

用 Fogus 在“The Joy of Clojure”或 Baker 在 1993 年发表的论文中的近似说法：如果任何两个可变对象现在解析为相等，并不能保证它们之后还将相等。如果两个对象永远不相等，那么它们在技术上就永远不相等;-)

### 2.6. SHOULD NOT declare abstract "var" members

> 不应该定义抽象的`var`成员

在抽象类或特质中声明抽象变量是一种不好的做法。请勿这样做：
```scala
trait Foo {
  var value: String
}
```

相反，我们更倾向于将抽象事物声明为 `def`：
```scala
trait Foo {
  def value: String
}

// 然后可以被覆盖为任何东西
class Bar(val value: String) extends Foo
```

原因与强制的限制有关 —— `var` 只能被 `var` 覆盖，允许在继承上自由选择的方法是对抽象成员使用 `def` 定义。为什么要对那些从接口继承的变量施加限制呢？`def` 是通用的，因此请使用它。

### 2.7. MUST NOT throw exceptions for validations of user input or flow control

> 不要在验证用户输入或流控制时抛出异常

理由有二:
1. 它违背了结构化程序设计的原则，因为一个例程最终会有多个退出点，因此更难推理 —— 堆栈展开的发生是一个可怕的、通常不可预测的副作用。
2. 异常没有记录在函数的签名中 —— Java 试图通过检查异常的概念来解决这个问题，但在实践中却非常糟糕，因为人们直接忽略了它们。

异常只对一件事有用 —— 向堆栈传递意外错误（bug）的信号，这样监督者就能捕捉到这些错误，并决定采取一些措施，如记录错误、发送通知、重启有问题的组件等。

作为对权威的呼吁，请参考 [Functional Programming with Scala](http://www.manning.com/bjarnason/) 第 4 章。

### 2.8. MUST NOT catch Throwable when catching Exceptions

> 捕获 `Exception` 时，不要捕获 `Throwable`

永远、永远、永远不要这样做：

```scala
try {
 something()
} catch {
 case ex: Throwable =>
   blaBla()
}
```

永远不要捕获 `Throwable`，因为我们谈论的可能是永远不应该被捕获的极其致命的异常，这可能会导致进程崩溃。 例如，如果 JVM 抛出内存不足的错误，即使在 `catch` 子句中重新抛出异常，也可能为时已晚。考虑到进程内存不足，垃圾收集器可能会接管并冻结一切，使进程以不可恢复的僵尸状态结束。这意味着外部监督程序(如 Upstart）将没有机会重启它。

请这样做：

```scala
import scala.util.control.NonFatal

try {
 something()
} catch {
 case NonFatal(ex) =>
   blaBla()
}
```

### 2.9. MUST NOT use "null"

> 不要使用 `null`

必须避免使用 `null`。请使用 Scala 的 `Option[T]`。空值很容易出错，因为编译器无法保护你。函数定义中出现的可空值不会在这些定义中记录。因此要避免这样做：
```scala
def hello(name: String) =
  if (name != null)
    println(s"Hello, $name")
  else
    println("Hello, anonymous")
```

第一步，你可以这样做：

```scala
def hello(name: Option[String]) = {
  val n = name.getOrElse("anonymous")
  println(s"Hello, $n")
}
```

使用 `Option[T]` 的意义在于，编译器会强迫你以某种方式处理它：
1. 你要么必须立即处理它（例如，通过提供默认值、抛出异常等。。。）
2. 或者你可以将 `Option` 结果向调用栈传播

还要记住，`Option` 就像一个拥有0或1个元素的集合，所以你可以使用 `foreach`，这完全是惯用的：
```scala
val name: Option[String] = ???

for (n <- name) {
  // 仅在name不为None的情况下执行
  println(n)
}
```

将多个 `Option` 组合起来也很容易：
```scala
val name: Option[String] = ???
val age: Option[Int] = ???

for (n <- name; a <- age)
  println(s"Name: $n, age: $a")
```

由于 `Option` 也被视为 `Iterable`，因此你可以在集合上使用 `flatMap` 来摆脱 `None` 值：
```scala
val list = Seq(1,2,3,4,5,6)

list.flatMap(x => Some(x).filter(_ % 2 == 0))
// => 2,4,6
```

### 2.10. MUST NOT use `Option.get`

> 不要使用 `Option.get`

你可能会想这样做：

```scala
val someValue: Option[Double] = ???

// ....
val result = someValue.get + 1
```

永远不要这样做，因为你正在用 `NullPointerException` 转换为 `NoSuchElementException`，这违背了使用 `Option` 的初衷。

备择方案：

1. 使用 `Option.getOrElse`
2. 使用 `Option.fold`
3. 使用模式匹配并明确处理 `None` 分支
4. 不从可选上下文中提取值

以（4）为例，不从可选上下文中提取值的意思是这样：
```scala
val result = someValue.map(_ + 1)
```

### 2.11. MUST NOT use Java's Date or Calendar, instead use `java.time` (JSR-310)

> 不要使用 Java 的日期或日历，而是使用 `java.time` (JSR-310)

Java 标准库中的日期和日历类非常糟糕，因为：
1. 结果对象是可变的，这对于表达日期没有意义，日期应该是一个值（如果你必须在有字符串的地方使用 `StringBuffer`，你会有什么感觉？）
2. 月份编号从零开始
3. 特别是日期不保留时区信息，因此日期值完全无用
4. GMT 和 UTC 之间没有区别
5. 年份表示为 2 位数字而不是 4 位

相反，请始终使用 Java 8 中引入的 [`java.time`](https://docs.oracle.com/javase/8/docs/api/java/time/package-summary.html) API - 或者如果你陷入 Java 8 之前的领域，请使用 [`Joda-Time`](http://www.joda.org/joda-time/)，它是 `java.time` 的精神祖先。。


### 2.12. SHOULD NOT use Any or AnyRef or isInstanceOf / asInstanceOf

> 不应该使用 `Any` 或 `AnyRef` 或 `isInstanceOf/asInstanceOf`

Avoid using Any or AnyRef or explicit casting, unless you've got a
really good reason for it. Scala is a language that derives value from
its expressive type system, usage of Any or of typecasting represents
a hole in this expressive type system and the compiler doesn't know
how to help you there. In general, something like this is bad:

```scala
val json: Any = ???

if (json.isInstanceOf[String])
  doSomethingWithString(json.asInstanceOf[String])
else if (json.isInstanceOf[Map])
  doSomethingWithMap(json.asInstanceOf[Map])
else
  ???
```

Often we are using Any when doing deserialization. Instead of working
with Any, think about the generic type you want and the set of
sub-types you need, and come up with an Algebraic Data-Type:

```scala
sealed trait JsValue

case class JsNumber(v: Double) extends JsValue
case class JsBool(v: Boolean) extends JsValue
case class JsString(v: String) extends JsValue
case class JsObject(map: Map[String, JsValue]) extends JsValue
case class JsArray(list: Seq[JsValue]) extends JsValue
case object JsNull extends JsValue
```

Now, instead of operating on Any, we can do pattern matching on
JsValue and the compiler can help us here on missing branches, since
the choice is finite. This will trigger a warning on missing branches:

```scala
val json: JsValue = ???
json match {
  case JsString(v) => doSomethingWithString(v)
  case JsNumber(v) => doSomethingWithNumber(v)
  // ...
}
```

### 2.13. MUST serialize dates as either Unix timestamp, or as ISO 8601

> 必须将日期序列化为 Unix 时间戳或 ISO 8601

Unix timestamps, provided that we are talking about the number of
seconds or milliseconds since 1970-01-01 00:00:00 UTC (with emphasis
on UTC) are a decent cross-platform serialization format. It does have
the disadvantage that it has limits in what it can express. ISO-8601
is a decent serialization format supported by most libraries.

Avoid anything else and also when storing dates without a timezone
attached (like in MySQL), always express that info in UTC.

### 2.14. MUST NOT use magic values

> 不要使用魔法值

Although not uncommon in other languages to use "magic" (special)
values like `-1` to signal particular outcomes, in Scala there are a
range of types to make intent clear. `Option`, `Either`, `Try` are
examples. Also, in case you want to express more than a boolean
success or failure, you can always come up with an algebraic data
type.

Don't do this:

```scala
val index = list.find(someTest).getOrElse(-1)
```

### 2.15. SHOULD NOT use "var" as shared state

> 不应该使用`var`作为共享状态

Avoid using "var" at least when speaking about shared mutable
state. Because if you do have shared state expressed as vars, you'd
better synchronize it and it gets ugly fast. Much better is to avoid
it. In case you really need mutable shared state, use an atomic
reference and store immutable things in it. Also checkout
[Scala-STM](https://nbronson.github.io/scala-stm/).

So instead of something like this:

```scala
class Something {
  private var cache = Map.empty[String, String]
}
```

If you can't really avoid that variable, prefer doing this:

```scala
import java.util.concurrent.atomic._

class Something {
  private val cache =
    new AtomicReference(Map.empty[String, String])
}
```

Yes, it introduces overhead due to the synchronization required, which
in the case of an atomic reference means spin loops. But it will save
you from lots and lots of headaches later. And it's best to avoid
mutation entirely.

### 2.16. Public functions SHOULD have an explicit return type

> 公共函数应该具有显式的返回类型

Prefer this:

```scala
def someFunction(param1: T1, param2: T2): Result = {
  ???
}
```

To this:

```scala
def someFunction(param1: T1, param2: T2) = {
  ???
}
```

Yeah, type inference on the result of a function is great and all, but
for public methods:

1. it's not OK to rely on an IDE or to inspect the implementation in
   order to see the returned type
2. Scala currently infers the most specialized type possible, because
   in Scala the return type on functions is covariant, so you might
   actually get a really ugly type back

For example, what is the returned type of this function:

```scala
def sayHelloRunnable(name: String) = new Runnable {
  def sayIt() = println(s"Hello, $name")
  def run() = sayIt()
}
```

Do you think it's `Runnable`?
Wrong, it's `Runnable{def sayIt(): Unit}`.

As a side-effect, this also increases compilation times, as whenever
`sayHelloRunnable` changes implementation, it also changes the
signature so everything that depends on it must be recompiled.

### 2.17. SHOULD NOT define case classes nested in other classes

> 不应该定义嵌套在其他类中的样例类

It is tempting, but you should almost never define nested case classes
inside another object/class because it messes with Java's
serialization. The reason is that when you serialize a case class it
closes over the "this" pointer and serializes the whole object, which
if you are putting in your App object means for every instance of a
case class you serialize the whole world.

And the thing with case classes specifically is that:

1. one expects a case class to be immutable (a value, a fact) and hence
2. one expects a case class to be easily serializable

Prefer flat hierachies.

### 2.18 MUST NOT include classes, traits and objects inside package objects

> 不要在包对象中包含类、特质和对象

Classes, including case classes, traits and objects do not belong
inside package objects. It is unnecessary, confuses the compiler and
is therefore discouraged. For example, refrain from doing the
following:

```scala
package foo

package object bar {
  case object FooBar
}
```

The same effect is achieved if all artifacts are inside a plain package:

```scala
package foo.bar

case object FooBar
```

Package objects should only contain value, method and type alias
definitions, etc.  Scala allows multiple public classes in a single
file, and the convention is to have the first letter of the filename
be lowercase in such cases.

#### Implicit value classes can be defined in a package object

In one rare circumstance, it makes sense to include classes defined directly in a `package object`. The reason for that is that implicit classes need to be nested inside another object/class, you cannot define a top level implicit in Scala. Nesting implicit value classes inside a `package object` also allows us to create a great importing experience for a library, as the single import of the package object will bring in all the necessary implicits.

There is also no way around this, as defining the implicit value class means we cannot effectively define the implicit and the class separately, the whole point is to let the compiler avoid the runtime boxing by generating all the "right code" at compile time, by predicting the runtime boxing. This is the optimal way to achieve a pimp my library pattern performance wise, and we need the entire syntax "in one place".

```scala

package object dsl {
  implicit class DateTimeAugmenter(val date: Datetime) extends AnyVal {
    def yesterday: DateTime = date.plusDays(-1)
  }
}

```

### 2.19 SHOULD use head/tail and init/last decomposition only if they can be done in constant time and memory

> 应该仅在head/tail和init/last分解可以在常量时间和内存中完成时，才使用它们

Example of head/tail decomposition:

```scala
def recursiveSumList(numbers: List[Int], accumulator: Int): Int =
  numbers match {
    case Nil =>
      accumulator

    case head :: tail =>
      recursiveSumList(tail, accumulator + head)
  }
```

`List` has a special head/tail extractor `::` because `List`s are **made** by always appending an element to the front of the list:

```scala
val numbers = 1 :: 2 :: 3 :: Nil
```

This is the same as:

```scala
val numbers = Nil.::(3).::(2).::(1)
```

For this reason, both `head` and `tail` on a list need only constant time and memory! These operations are `O(1)`.

There is another head/tail extractor called `+:` that works on any `Seq`:

```scala
def recursiveSumSeq(numbers: Seq[Int], accumulator: Int): Int =
  numbers match {
    case Nil =>
      accumulator

    case head +: tail =>
      recursiveSumSeq(tail, accumulator + head)
  }
```

You can find the implementation of `+:` [here](https://github.com/scala/scala/blob/v2.12.4/src/library/scala/collection/SeqExtractors.scala). The problem is that other collections than `List` do not necessarily head/tail-decompose in constant time and memory, e.g. an `Array`:

```scala
val numbers = Array.range(0, 10000000)

recursiveSumSeq(numbers, 0)
```

This is highly inefficient: each `tail` on an `Array` takes `O(n)` time and memory, because every time a new array needs to be created!

Unfortunately, the Scala collections library permits these kinds of inefficient operations. We have to keep an eye out for them.

---

An example for an efficient init/last decomposition is `scala.collection.immutable.Queue`. It is backed by two `List`s and the efficiency of `head`, `tail`, `init` and `last` is *amortized constant* time and memory, as explained in the [Scala collection performance characteristics](http://docs.scala-lang.org/overviews/collections/performance-characteristics.html).

I don't think that init/last decomposition is all that common. In general, it is analogue to head/tail decomposition. The init/last deconstructor for any `Seq` is `:+`.


### 2.20 MUST NOT use `Seq.head`

> 使用`Seq.head`

You might be tempted to this:

```scala
val userList: List[User] = ???

// ....
val firstName = userList.head.firstName
```

Don't ever do this, as this will throw `NoSuchElementException` if the sequence is empty.

Alternatives:

1. using `Seq.headOption` possibly combined with `getOrElse` or pattern matching

    Example:
    
    ```scala
    val firstName = userList.headOption match {
        case Some(user) => user.firstName
        case _ => "Unknown"
      }
    ```

2. using pattern matching with the cons operator `::` if you're dealing with a `List`

    Example:
    
    ```scala
    val firstName = userList match {
     case head :: _ => head.firstName
     case _ => "Unknown"
    }
    ```

3. using `NonEmptyList` if it is required that the list should never be empty. (See [cats](https://typelevel.org/cats/datatypes/nel.html), [scalaz](https://github.com/scalaz/scalaz/blob/series/7.3.x/core/src/main/scala/scalaz/NonEmptyList.scala), ...)

### 2.21 Case classes SHOULD be final

> 样例类应该标记为`final`

Extending a case class will lead to unexpected behaviour. Observe the following:
```scala
scala> case class Foo(v:Int)
defined class Foo

scala> class Bar(v: Int, val x: Int) extends Foo(v)
defined class Bar

scala> new Bar(1, 1) == new Bar(1, 1)
res25: Boolean = true

scala> new Bar(1, 1) == new Bar(1, 2)
res26: Boolean = true
// ????
scala> new Bar(1,1) == Foo(1)
res27: Boolean = true

scala> class Baz(v: Int) extends Foo(v)
defined class Baz

scala> new Baz(1) == new Bar(1,1)
res29: Boolean = true //???

scala> println (new Bar(1,1))
Foo(1) // ???

scala> new Bar(1,2).copy()
res49: Foo = Foo(1) // ???
```
Credits:[Why case classes should be final](https://stackoverflow.com/a/34562046/3856808)

So by default case classes should always be defined as final. 

Example:

```scala
final case class User(name: String, id: Long)
```

### 2.22 SHOULD NOT use `scala.App`

> 不应该使用`scala.App`

`scala.App` is often used to denote the entrypoint of the application:

```scala
object HelloWorldApp extends App {
  println("hello, world!")
}
```

`DelayedInit` one of the mechanisms used to implement `scala.App` [has been deprecated](https://github.com/scala/scala/pull/3563).
Any variables defined in the object body will be available as fields, unless the `private` access modifier is applied.
Prefer the simpler alternative of defining a main method:

```scala
object HelloWorldApp {
  def main(args: Array[String]): Unit = println("hello, world!")
}
```
