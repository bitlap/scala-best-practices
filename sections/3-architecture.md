## 3. Application Architecture

<img src=".././assets/scala-logo-256.png"  align="right" width="128" height="128" />

### 3.1. SHOULD NOT use the Cake Pattern

> 不应该使用 Cake Pattern

[从理论上讲](https://www.youtube.com/watch?v=yLbdw06tKPQ)，Cake 模式是一个非常好的想法 —— 使用特质作为可以组合的模块，让你能够覆盖 `import`，而编译时依赖注入是一个副作用。

实际上，我所见过的所有 Cake 实现都非常糟糕，因此新项目应该远离 Cake，现有项目也应该从 Cake 上迁移。

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

但这并不是唯一的问题。更大的问题是，开发人员都很懒惰，所以你最终得到的是有很多依赖关系和职责的大型组件，因为 Cake 鼓励这样做。
在造成这种破坏的原始开发人员离开项目之后，你最终会得到其他更小的组件，这些组件复制了原始组件的功能，只是因为原始组件非常难以测试，因为你必须模拟或存根太多东西（另一种 *代码气味*）。
这样你就陷入了一个永远重复的循环，开发人员最终讨厌代码库，只做了完成任务所需的最少的工作，最终得到了庞大、丑陋且从根本上有缺陷的组件。
而且由于 Cake 天生导致的紧密耦合，它们并不容易被重构。

那么，当这样的东西更容易阅读也更符合常识时，为什么还要像上面那样做呢：
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
如果一个组件依赖于难以初始化的参数，那便是 *代码气味*。如果为了测试纯粹的业务逻辑而需要在测试中模拟或存根接口，这很可能便是 *代码气味*；-)

不要把痛苦的事情藏在地毯下，而是要解决它。

### 3.2. MUST NOT put things in Play's Global

> 禁止将内容放入 Play 的 Global

这种情况我见了一次又一次。

各位，Play 的 [Global](https://www.playframework.com/documentation/2.3.x/ScalaGlobal) 并不是一个可以把你的零散代码塞进去的桶。它的目的是与 Play 的配置和生命周期挂钩，仅此而已。

为你的实用程序创建一个自己的独立命名空间。

### 3.3. SHOULD NOT apply optimizations without profiling

> 没有经过分析不应该优化

分析是进行优化的先决条件。除非通过分析发现瓶颈，否则从不优化。

这是因为我们对系统行为方式的直觉经常会失灵，而且，在没有确凿数据的情况下进行优化，可能会产生多种影响：
- 可能会使代码或架构复杂化，从而更难在全局范围内应用后续优化
- 你的工作可能会白费，或者实际上会导致更多的性能下降

有多种策略可供选择，你最好全部采用：
- 一个好的探查器可以告诉你一些不明显的瓶颈，我最喜欢的是YourKit探查器，但Oracle的VisualVM是免费的，通常也足够好了。
- 从运行中的生产系统中收集度量指标，通过 [Dropwizard Metrics](https://dropwizard.github.io/metrics/3.1.0/) 等库，并将其推送到 [Graphite](http://graphite.wikidot.com/) 之类的软件中，这种策略可引导你朝着正确的方向前进
- 通过编写基准代码来比较解决方案。但要注意的是，基准测试并不容易，你至少应该使用一个库，诸如 [JMH](http://openjdk.java.net/projects/code-tools/jmh/)、[Scala Meter](https://scalameter.github.io/)

总的来说 - 要测量，不要猜测。

### 3.4. SHOULD be mindful of the garbage collector

> 应注意垃圾回收器

除非有必要，否则不要过度分配资源。我们希望避免微优化，但始终要注意分配资源对系统的影响。

用 [马丁-汤姆森（Martin Thomson）](http://www.infoq.com/presentations/top-10-performance-myths) 的话说：
如果给垃圾收集器施加压力，就会增加 “stop-the-world” 冻结的延迟时间和发生次数，垃圾收集器就会像 GIL 一样发挥作用，从而限制性能和纵向可扩展性。

示例：

```scala
query.filter(_.someField.inSet(Set(name)))
```

这是我们项目中出现的一个示例，原因是 Slick 的 API 存在问题。因此，开发人员没有进行 `===` 测试，而是选择了对一个元素序列进行 `inSet` 操作。
这种分配 1 个元素的集合会在每次方法调用时发生。这可不好，能避免的就应该避免。

另一个例子：

```scala
someCollection
 .filter(Set(a,b,c).contains)
 .map(_.name)
```

首先，每次都会在我们集合的每个元素上创建一个 `Set` 集合。其次，`filter` 和 `map` 可以压缩在一起操作，否则我们最终会产生更多的垃圾，并花费更多的时间构建最终集合：
```scala
val isIDValid = Set(a,b,c)

someCollection.collect {
  case x if isIDValid(x) => x.name
}
```

一个经常出现的通用示例，举例说明了可以压缩的无用遍历和运算符：
```scala
collection
  .filter(bySomething)
  .map(toSomethingElse)
  .filter(again)
  .headOption
```

此外，请注意你的需求，并使用适合你的使用情况的数据结构。
你想建立一个栈？那就用 `List`。要为列表建立索引？那就是 `Vector`。
要追加到列表的末尾？那也是一个 `Vector`。
要 push 到前面并从后面 pull 出来？这是一个 `Queue`。
你有一组事物，并想检查其成员资格？这是一个 `Set`。
你有一个要保持有序的列表？那就是 `SortedSet`。
这不是高深的科学，而只是计算机科学 101。

我们不是在讨论极端的微优化，我们甚至不是在讨论 Scala、FP 或 JVM 特有的东西。
但请注意你正在做的事情，尽量不要做不必要的分配，因为日后修复起来要困难得多。

顺便说一下，有一个显而易见的解决方案可以在进行 `filter` 和 `map` 的同时保持表现力 —— 惰性集合，这在 Scala 中意味着，如果需要记忆化则指 [Stream](http://www.scala-lang.org/api/current/index.html#scala.collection.immutable.Stream)， 如果不需要记忆化则指 [Iterable](http://docs.oracle.com/javase/7/docs/api/java/lang/Iterable.html)。

此外，请务必阅读 [Rule 3.3](#33-should-not-apply-optimizations-without-profiling)。

### 3.5. MUST NOT use parameterless ConfigFactory.load() or access a Config object directly

> 禁止使用无参数的 `ConfigFactory.load()` 或直接访问 `Config` 对象

每当需要从配置中调用某些内容时，调用无参数可用的 `ConfigFactory.load()` 方法，但这样做会适得其反，例如在编写测试时。

如果你的类中到处都有 [`ConfigFactory.load()`](https://typesafehub.github.io/config/latest/api/com/typesafe/config/ConfigFactory.html#load--)，它们基本上就是在代码运行时加载默认配置，而在测试环境中，你需要加载修改过的配置（如不同的超时、不同的实现、不同的 IP 等），这往往不是你真正想要的。

千万不要这么做：
```scala
class MyComponent {
  private val ip = ConfigFactory.load().getString("myComponent.ip")
}
```

处理它的一种方法是将 `Config` 实例本身传递给需要它的人，或将其中的所需值传递给需要它的人。
这里描述的情况实际上是 [prefer dependency injection (DI) over Service Locator](http://stackoverflow.com/questions/1638919/how-to-explain-dependency-injection-to-a-5-year-old/1638961#1638961) 的一种做法。

你可以调用 `ConfigFactory.load()`，但要从应用程序的根目录调用，例如在你的 `main()`（或类似的地方）中调用，这样你就不必硬编码配置的文件名。

另一种好的做法是建立特定领域的配置类、这些类是从通用的、类似于 map 的 `Config` 对象解析而来。这种方法的好处是，专门的配置类能忠实地代表你的特定配置需求，并且一旦解析完毕，就能以类型安全的方式处理编译后的类。（这里的 “更安全” 指的是使用 `config.ip` 而不是 `config.getString("ip")`）。

这样做的另一个好处是代码更清晰，因为特定领域的配置类以更明确和可读的方式传达所需的属性。

请看下面的示例：

```scala
/** 
 * 这是特定的领域配置类，具有一组预定义的属性，而不是类似map的属性包。
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

这种方法的优点：
- 配置对象只是不可变的样例类，可以很容易地实例化
- 你的组件最终将依赖于只与它们相关的具体且类型安全的配置定义，而不是接收一个包含所有内容且实例化成本高昂的单一且不安全的 `Config`
- 现在，你的集成开发环境可以在文档和可发现性方面提供帮助
- 而你的编译器可以帮助解决拼写错误

关于样式的注意事项：这些配置样例类往往会变得很大，并且包含多种类型（如 `Int`、`String` 等），因此与依赖位置的相比，使用命名参数会使代码更不易更改，更不易出错。
这里选择的缩进方式使实例化看起来像一个 `Map` 或 JSON 对象。