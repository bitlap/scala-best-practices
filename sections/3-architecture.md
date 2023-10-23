## 3. Application Architecture

<img src=".././assets/scala-logo-256.png"  align="right" width="128" height="128" />

### 3.1. SHOULD NOT use the Cake Pattern

> 不应该使用 Cake Pattern

[从理论上讲](https://www.youtube.com/watch?v=yLbdw06tKPQ)，Cake 模式是一个非常好的想法 —— 使用特质作为可以组合的模块，让你能够覆盖 `import`，而编译时依赖项注入是一个副作用。

实际上，我所见过的所有 Cake 实现都非常糟糕，新项目应该远离 Cake，现有项目也应该从 Cake 上迁移。

由于 Cake 是一种鲜为人知的设计模式，因此人们并没有正确地实现它。我还没见过把特质设计成抽象模块的 Cake 实现，也没见过对生命周期问题给予适当关注的 Cake 实现。
在实践中出现的情况就是马虎了事，结果导致一团乱麻。
Scala 允许你实现 Cake 模式这样的功能，这很了不起，凸显了 OOP 的真正威力，但这并不意味着你应该这样做，因为如果你的目的是在不同组件之间进行依赖注入和解耦，那么你将彻底失败，并将维护负担强加给你的同事。

例如，这是 Cake 中常见的情况：
```scala
trait SomeServiceComponent {
  type SomeService <: SomeServiceLike
  val someService: SomeService // abstract

  trait SomeServiceLike {
    def query: Rows
  }
}

trait SomeServiceComponentImpl extends SomeServiceComponent {
  self: DBServiceComponent =>

  val someService = new SomeService

  class SomeService extends SomeServiceLike {
    def query = dbService.query
  }
}
```

在上面的示例中，`someService` 实际上是一个真正的[单例](https://en.wikipedia.org/wiki/Singleton_pattern)，因为它可能缺少*生命周期管理*。
如果读完这段代码后，你还没有被单例缺少生命周期管理所触动，那么请了解大多数 Cake 实现的丑陋秘密。对于那些有意识地正确执行此操作的少数人来说，他们最终会陷入 JVM 初始化地狱。

但这并不是唯一的问题。更大的问题是，开发人员都很懒惰，所以你最终得到的是有很多依赖关系和责任的大型组件，因为 Cake 鼓励这样做。
在造成这种破坏的原始开发人员离开项目之后，你最终会得到其他更小的组件，这些组件复制了原始组件的功能，只是因为原始组件非常难以测试，因为你必须模拟或存根太多东西（另一种 *代码气味*）。
这样你就陷入了一个永远重复的循环，开发人员最终讨厌代码库，只做了完成任务所需的最少的工作，最终得到了其他庞大、丑陋且从根本上有缺陷的组件。
而且由于 Cake 自然导致的紧密耦合，它们不容易重构。

那么，当这样的东西更容易阅读和也更符合常识时，为什么要像上面那样做呢：
```scala
class SomeService(dbService: DBService) {
  def query = dbService.query
}
```

或者，如果你真的需要抽象的东西（但请阅读第 [2.4 条](2-language-rules.md#24-should-not-define-useless-traits) 关于不定义无用特质的规定）：
```scala
trait SomeService {
  def query: Rows
}

object SomeService {
  /** Builder for [[SomeService]] */
  def apply(dbService: DBService): SomeService =
    new SomeServiceImpl(dbService)

  private final class SomeServiceImpl(dbService: DBService)
    extends SomeService {
    def query: Rows = dbService.query
  }
}
```

你的依赖关系是否变得疯狂？那些构造函数是不是开始疼了？这是一个特点。
它被称为 “*痛苦驱动开发*”（简称 PDD :-)）。这表明架构出现了问题，而各种依赖注入库或 Cake 模式并不是在解决问题，而是通过将垃圾隐藏在地毯下面来解决症状。

所以，还是使用简单可靠的 *构造函数参数* 吧。如果你确实需要使用依赖注入库，那就在边缘使用（比如 Play 的控制器）。因为如果一个组件依赖太多东西，就会产生 *代码气味*。
如果一个组件依赖于难以初始化的参数，那就是 *代码气味*。如果为了测试纯粹的业务逻辑而需要在测试中模拟或存根接口，这很可能就是 *代码气味*；-)

不要把痛苦的事情藏在地毯下，而是要解决它。

### 3.2. MUST NOT put things in Play's Global

这种情况我见了一次又一次。

各位，Play 的 [Global](https://www.playframework.com/documentation/2.3.x/ScalaGlobal) 并不是一个可以把你的零散代码塞进去的桶。它的目的是与 Play 的配置和生命周期挂钩，仅此而已。

为你的实用程序创建一个自己的独立命名空间。

### 3.3. SHOULD NOT apply optimizations without profiling

Profiling is a prerequisite for doing optimizations. Never work on
optimizations, unless through profiling you discover the actual
bottlenecks.

This is because our intuition about how the system behaves often fails
us and multiple effects could happen by applying optimizations without
having hard numbers:

- you could complicate the code or the architecture, thus making it
  harder to apply later optimizations globally
- your work could be in vain or it could actually lead to more
  performance degradation

Multiple strategies available and you should preferably do all of
them:

- a good profiler can tell you about bottlenecks that aren't obvious,
  my favorite being YourKit Profiler, but Oracle's VisualVM is free
  and often good enough
- collect metrics from the running production systems, by means of a
  library such as
  [Dropwizard Metrics](https://dropwizard.github.io/metrics/3.1.0/)
  and push them in something like
  [Graphite](http://graphite.wikidot.com/), a strategy that can lead
  you in the right direction
- compare solutions by writing benchmarking code, but note that
  benchmarking is not easy and you should at least use a library like
  [JMH](http://openjdk.java.net/projects/code-tools/jmh/), [Scala Meter](https://scalameter.github.io/)

Overall - measure, don't guess.

### 3.4. SHOULD be mindful of the garbage collector

Don't over allocate resources, unless you need to. We want to avoid
micro optimizations, but always be mindful about the effects
allocations can have on your system.

In the
[words of Martin Thomson](http://www.infoq.com/presentations/top-10-performance-myths),
if you stress the garbage collector, you'll increase the latency on
stop-the-world freezes and the number of such occurrences, with the
garbage collector acting like a GIL and thus limiting performance and
vertical scalability.

Example:

```scala
query.filter(_.someField.inSet(Set(name)))
```

This is a sample that occurred in our project due to a problem with
Slick's API. So instead of a `===` test, the developer chose to do an
`inSet` operation with a sequence of 1 element. This allocation of a
collection of 1 element happens on every method call. Now that's not
good, what can be avoided should be avoided.

Another example:

```scala
someCollection
 .filter(Set(a,b,c).contains)
 .map(_.name)
```

First of all, this creates a Set every single time, on each element of
our collection. Second of all, filter and map can be compressed in one
operation, otherwise we end up with more garbage and more time spent
building the final collection:

```scala
val isIDValid = Set(a,b,c)

someCollection.collect {
  case x if isIDValid(x) => x.name
}
```

A generic example that often pops up, exemplifying useless traversals
and operators that could be compressed:

```scala
collection
  .filter(bySomething)
  .map(toSomethingElse)
  .filter(again)
  .headOption
```

Also, take notice of your requirements and use the data-structure
suitable for your use-case. You want to build a stack? That's a
`List`. You want to index a list? That's a `Vector`. You want to
append to the end of a list? That's again a `Vector`. You want to push
to the front and pull from the back? That's a `Queue`. You have a set
of things and want to check for membership? That's a `Set`. You have a
list of things that you want to keep ordered? That's a
`SortedSet`. This isn't rocket science, just computer science 101.

We are not talking about extreme micro optimizations here, we aren't
even talking about something that's Scala, or FP, or JVM specific
here, but be mindful of what you're doing and try to not do
unnecessary allocations, as it's much harder fixing it later.

BTW, there is an obvious solution for keeping expressiveness while
doing filtering and mapping - lazy collections, which in Scala means
[Stream](http://www.scala-lang.org/api/current/index.html#scala.collection.immutable.Stream)
if you need memoization or
[Iterable](http://docs.oracle.com/javase/7/docs/api/java/lang/Iterable.html)
if you don't need memoization.

Also, make sure to read the
[Rule 3.3](#33-should-not-apply-optimizations-without-profiling) on
profiling.

### 3.5. MUST NOT use parameterless ConfigFactory.load() or access a Config object directly

It may be very tempting to call the oh-so-available-and-parameterless
`ConfigFactory.load()` method whenever you need to pull something from
the configuration, but doing so will boomerang back at you, for
instance when writing tests.

If you have
[`ConfigFactory.load()`](https://typesafehub.github.io/config/latest/api/com/typesafe/config/ConfigFactory.html#load--)
scattered all around your classes they are basically loading the
default configuration when your code runs, which more often than not,
is not what you really want to happen in a testing environment, where
you need to have a modified configuration loaded (e.g., different
timeouts, different implementations, different IPs, etc.).

NEVER do this:

```scala
class MyComponent {
  private val ip = ConfigFactory.load().getString("myComponent.ip")
}
```

One way to go about dealing with it, is to pass the `Config` instance
itself to whomever needs it, or have the needed values from it passed
in. The situation described here is in fact a flavor of the
[prefer dependency injection (DI) over Service Locator](http://stackoverflow.com/questions/1638919/how-to-explain-dependency-injection-to-a-5-year-old/1638961#1638961)
practice.

You can call `ConfigFactory.load()`, but from your application's root,
say in your `main()` (or equivalent) so that you don't have to hardcode your
configuration's filename.

Another good practice is to have domain specific config classes,
which are parsed from the general purpose, map-like, Config
objects. The benefit of this approach is that specialized config
classes faithfully represent your specific configuration needs, and
once parsed, allow you to work against compiled classes in a more
type-safe way (where "safer" means you do `config.ip`, instead of
`config.getString("ip")`).

This also has the benefit of clarity, as your domain specific config
class conveys the needed properties in a more explicit and readable
manner.

Consider the following example:

```scala
/** This is your domain specific config class, with a pre-defined set of
  * properties you've modeled according to your domain, as opposed to
  * a map-like properties bag
  */
case class AppConfig(
  myComponent: MyComponentConfig,
  httpClient: HttpClientConfig
)

/** Configuration for [[MyComponent]] */
case class MyComponentConfig(ip: String)

/** Configuration for [[HttpClient]] */
case class HttpClientConfig(
  requestTimeout: FiniteDuration,
  maxConnectionsPerHost: Int
)

object AppConfig {
  /** Loads your config.
    * To be used from `main()` or equivalent.
    */
  def loadFromEnvironment(): AppConfig =
    load(ConfigUtil.loadFromEnvironment())

  /** Load from a given Typesafe Config object */
  def load(config: Config): AppConfig =
    AppConfig(
        myComponent = MyComponentConfig(
          ip = config.getString("myComponent.ip")
        ),
        httpClient = HttpClientConfig(
          requestTimeout = config.getDuration("httpClient.requestTimeout", TimeUnit.MILLISECONDS).millis,
          maxConnectionsPerHost = config.getInt("httpClient.maxConnectionsPerHost")
        )
    )
}

object ConfigUtil {
  /** Utility to replace direct usage of ConfigFactory.load() */
  def loadFromEnvironment(): Config = {
    Option(System.getProperty("config.file"))
      .map(f => ConfigFactory.parseFile(f).resolve())
      .getOrElse(
        ConfigFactory.load(System.getProperty(
          "config.resource", "application.conf")))
  }
}

/** One component */
class HttpClient(config: HttpClientConfig) {
  ???
}

/** Another component, depending on your domain specific config.
  * Also notice the sane dependency injection ;-)
  */
class MyComponent(config: MyComponentConfig, httpClient: HttpClient) {
  ???
}
```

Benefits of this approach:

- the config objects are just immutable case classes of primitives that
  can be easily instantiated
- your components end up depending on concrete and type-safe configuration
  definitions related only to them, instead of receiving a monolithic and
  unsafe `Config` that contains everything and that's expensive to instantiate
- and now your IDE can help with documentation and discoverability
- and your compiler can help with spelling errors

NOTE about style: these configuration case classes tend to get big and to contain
primitives (e.g. ints, strings, etc.), so usage of named
parameters makes the code more resistant to change and less error-prone,
versus relying on positioning. The style of indentation chosen here makes
the instantiation look like a `Map` or a JSON object if you want.
