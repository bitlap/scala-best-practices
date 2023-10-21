## 1. Hygienic Rules

<img src=".././assets/scala-logo-256.png"  align="right" width="128" height="128" />

这些是超越语言或平台的通用卫生规则。编程语言是一种针对计算机系统的交流形式，也包括与你的同事和未来的自己交流，所以尊重这些规则就像上完厕所后洗手一样。

### 1.1. SHOULD enforce a reasonable line length

> 应该强制实行合理的代码行长度

关于排版有一门完整的科学表明，当文本行太宽时，人们会失去注意力，一行长的文字会让人很难判断该行的开始或结束位置，并且很难继续下一行，因为你的眼睛必须从右到左移动很多。这也使得扫描重要细节变得困难。

在排版中，最佳行长被认为是 50 到 70 个字符之间。

在编程中，我们有缩进，所以规定行的长度为 60 字符是不可行的。80 字符通常是可以接受的，但在 Scala 中却不行，因为在 Scala 中我们使用了大量的闭包，如果你希望函数和类的名称又长又有描述性，那么 80 字符就太短了。

另一方面，IntelliJ IDEA 的默认设置为 120 个字符，这可能太宽了。是的，我知道我们使用的是 16:9 宽屏显示器，但这对可读性没有任何帮助，而如果行数较短，我们就可以在进行并排对比差异时充分利用这些宽屏显示器。如果行数较长，我们就需要花费精力去注意行尾的重要细节。

因此，作为一种平衡：
- 争取以 80 个字符作为软限制，如果变得难看
- 那么 100 字符就足够了，除了。。。
- 函数签名，如果有限制，可能会非常难看

另一方面，任何超过 120 个字符的内容都是令人憎恶的。


### 1.2. MUST NOT rely on a SBT or IDE plugin to do the formatting for you

> 不要依靠 SBT 或 IDE 插件进行代码格式化 

集成开发环境和 SBT 插件可以提供很大的帮助，但如果你想使用它们来自动格式化你的代码，那就要小心了。

你不会找到一个能够推断开发者意图的插件，因为这需要人类对代码的理解，几乎不可能做到。适当缩进和格式化的目的并不是为了遵循某些以事物崇拜的方式给你设定的僵硬规则，而是为了让代码更合理、更易读、更平易近人。缩进实际上是一门艺术，这并不可怕，因为你只需要对糟糕的代码有敏锐的嗅觉和修复的冲动。而开发人员的职责就是确保自己的代码不发臭。

因此，自动的方式是可以的，但要注意不要破坏别人精心格式化的代码，否则我要用散文给你一记耳光（otherwise I'll slap you in prose.应该是玩笑话或俚语）。

让我们想一想我说过的话 —— 如果代码行太长，插件应该如何破解？让我们来谈谈这一行（真实代码）：
```scala
    val dp = new DispatchPlan(new Set(filteredAssets), start = startDate, end = endDate, product, scheduleMap, availabilityMap, Set(activationIntervals.get), contractRepository, priceRepository)
```

在大多数情况下，插件只会进行截断，我在实践中见过很多这样的情况：
```scala
    val dp = new DispatchPlan(Set(filteredAssets), start =
      startDate, end = endDate, product, scheduleMap, availabilityMap,
      Set(activationIntervals), contractRepository, priceRepository)
```

这可看不懂，对吧？我的意思是，说真的，这看起来就像呕吐物。这正是我所看到的依赖插件工作的人们所产生的结果。我们也可以有这样的版本：
```scala
    val dp = new DispatchPlan(
      Set(filteredAssets),
      startDate,
      endDate,
      product,
      scheduleMap,
      availabilityMap,
      Set(activationIntervals),
      contractRepository,
      priceRepository
    )
```

看起来好多了。但事实上，在其他情况下这样做并不好。比方说，我们有一行想要断开：
```scala
   val result = service.something(param1, param2, param3, param4).map(transform)
```

现在，无论你如何处理，将这些参数放在自己的行上都是非常糟糕的：
```scala
    // 太糟糕了，因为转换调用不可见
val result = service.something(
      param1,
      param2,
      param3,
      param4).map(transform)

    // 很糟糕，因为它破坏了逻辑流程
    val result = service.something(
      param1,
      param2,
      param3,
      param4
    ).map(transform)
```

这样会好得多：

```scala
    val result = service
      .something(param1, param2, param3, param4)
      .map(transform)
```

现在好多了，不是吗？当然，有时调用太长，这样做并不合适。因此，你需要使用某种临时值，例如。。。
```scala
    val result = {
      val instance =
        object.something(
          myAwesomeParam1,
          otherParam2,
          someSeriousParam3,
          anEvenMoreSoParam4,
          lonelyParam5,
          catchSomeFn6,
          startDate7
        )

      for (x <- instance) yield
        transform(x)
    }
```

当然，有时如果代码非常糟糕，就需要进行重构 —— 比如，对于一个函数来说，也许参数太多了;-)

而且，我们谈论的仅仅是行的长度，一旦涉及到其他问题，情况就会变得更加复杂。所以，你真的找不到一个插件能做这样的分析，并能为你做出正确的决定。

### 1.3. SHOULD break long functions

> 应该避免长函数

理想情况下，函数应该只有几行。如果行数太多，我们就需要将其拆分成更小的函数，并为它们命名。

请注意，在 Scala 中，我们不一定必须在其他作用域中提供此类中间函数，这里的目的主要是为了提高可读性，因此在 Scala 中，我们可以使用内部函数将逻辑分解为多个部分。

### 1.4. MUST NOT introduce spelling errors in names and comments

> 不要在命名和注释中引入拼写错误

拼写错误非常烦人，会打断读者的阅读进度。使用拼写检查器。智能的集成开发环境具有内置的拼写检查器。请注意带下划线的拼写警告并修复它们。

### 1.5. Names MUST be meaningful

> 命名必须要有意义

*"计算机科学中只有两件事很难：缓存失效和命名。"* —— Phil Karlton

我们在这里有三条准则：
1. 给出描述性的名称，但不要过多，四个词已经太多了
2. 如果从上下文中可以很容易地推断出类型和用途，或者已经有了约定俗成的惯例，在命名时可以简明扼要
3. 如果采用描述性命名，不要说毫无意义的废话

举例来说，这样做是可以接受的：

```scala
for (p <- people) yield
  transformed(p)
```

我们可以从直接上下文中看出 `p` 是一个人，所以一个简短的单字母命名就可以了。这也是可以接受的，因为 `i` 是用作索引的既定约定：

```scala
for (i <- 0 until limit) yield ???
```

这通常是不可接受的，因为通常对于元组，集合的命名并不能很好地反映所包含的内容（如果你没有给这些元素一个名称，那么集合本身就会有一个糟糕的名称）：
```
someCollection.map(_._2)
```

另一方面，隐式参数可以使用简短的名称，因为它们是隐式传递的，我们不会在意它们，除非它们丢失了：
```scala
def query(id: Long)(implicit ec: ExecutionContext, c: WSClient): Future[Response]
```

这是不可取的，因为这个名称完全没有意义，即使有明显的描述意图：
```scala
def processItems(people: Seq[Person]) = ???
```

这是不可取的，因为这个函数的命名表示了一个副作用（`process` 是一个动词，表示处理/加工），但它并没有描述我们要对这些人做什么。`Items` 后缀毫无意义，因为我们可以使用 `processThingy`、`processRows`、`processStuff`，但表达的意思仍然是一样的 —— 什么都没有。它还会增加视觉上的混乱，因为字数越多，要阅读的文字就越多，而无意义的字只是噪音。

正确选择描述性名称 —— 好。废话连篇的名称 —— 不好。