## 4. Concurrency and Parallelism

<img src=".././assets/scala-logo-256.png"  align="right" width="128" height="128" />

### 4.1. SHOULD avoid concurrency like the plague it is

> 应该视并发如瘟疫

尽可能避免处理并发问题。善于处理并发问题的人会像躲避瘟疫一样躲避并发问题。

**警告：** 并发问题不仅会在讨论共享内存和线程时出现，而且在涉及任何类型的资源（如数据库）的争用时，进程之间也会发生。

例如，在 Linux 中使用 cron.d 计划每分钟执行一次作业，而该作业从 MySQL（表）中获取和更新项目时，该作业的执行时间可能会超过 1 分钟。
因此，你可能会看到 2 或 3 个进程同时执行，并在同一个 MySQL 表上竞争。

### 4.2. SHOULD use appropriate abstractions only where suitable - Future, Actors, Rx

> 应该只在适当的情况下使用适当的抽象

了解可用的抽象概念，并根据手头的任务在它们之间做出选择，总之，视手头的任务而定。
没有什么灵丹妙药可以普遍适用。抽象程度越高，其解决问题的范围就越小。但是，范围越小、功能越强，模型就越简单、可组合。
例如，Scala 社区中的许多开发人员都在过度使用 Akka Actors，这固然很好，但如果使用不当就会造成问题。
比如，在使用 `Future` 时，不要使用 Akka Actor。

> "*权力会导致腐败，绝对的权力会导致绝对的腐败*" -- Lord Acton

Scala 的 Futures 和 Promises 之所以好，是因为：
- 由于消除了并发性问题，它们本质上是可并行的
- 相当高效，默认情况下，因为将任务提交给隐式的 `ExecutionContext`时，该执行上下文可以在少数线程之间有效地进行多路复用（线程池中的线程数通常与 CPU 内核数成正比）
  成正比）
- 其他实现可能比 Scala 的标准 Future 更简单、更高效，因为标准 Future 的设计初衷是通用的
- 模型本身简单易用

Futures 和 Promises 之所以糟糕，是因为它们只向生产者和消费者传递一种 value（值） 信号，仅此而已。
如果你需要流或双向通信，Future 可能不是最好的抽象。

Akka Actors 的优点在于：
- 它们使异步边界上的双向通信变得简单 —— 例如，对于 WebSocket，Actors 便是最佳选择之一
- 使用 Akka 的 Actors，你可以轻松地为状态机建模（参见  `context.become`)
- 消息的处理有很强的非并发性保证 —— 消息是逐个处理的，因此在一个 actor 的上下文中无需担心并发问题
- 你可以直接使用 actor，而不是在内存中实现一个半吊子的队列来处理事情，因为它们的工作就是对消息进行排队并采取行动

Akka 的 Actors 很糟糕，因为：
- 对于许多任务来说，它们的级别都相当低
- 为保持大量状态的 actor 建模是非常容易的，最终得到的是一个无法横向扩展的系统
- 由于具有双向通信能力，数据流极易复杂到无法管理的地步
- 一般情况下，模型是 actor A 向 actor B 发送信息，但如果需要对事件流进行建模，A 和 B 之间的这种紧密耦合是不可接受的
- 在 Akka 的实际应用中，actor 往往会诱发不受控制的副作用，这不仅很容易出错，且与函数式编程相违背，它也不是 Scala 的习惯做法

Streaming abstractions such as [Play's Iteratees](https://www.playframework.com/documentation/2.5.x/Iteratees) / 
[Akka Streams](http://doc.akka.io/docs/akka-stream-and-http-experimental/2.0-M2/scala.html) /
[RxJava](https://github.com/ReactiveX/RxJava) /
[Reactive Streams](http://www.reactive-streams.org/) /
[FS2](https://github.com/functional-streams-for-scala/fs2) /
[Monix](https://github.com/alexandru/monix) are good because:

- they model unidirectional communications between producers and consumers 
- events flow in one direction and so you much easily transform and compose
  these streams
- depending on implementation, they address back-pressure concerns by
  default
- just as in the case of Future, because of the limitations, the model
  is simple to use and much more reasonable and composable than actors,
  with the exposed operators being awesome

Streams are bad because:

- they are only about unidirectional communications, it gets
  complicated if you want bidirectional communications, with 
  actors being better at having dialogs
- because of the strong contract they come with (i.e. no concurrent
  notifications for example), implementing new operators and data-source
  can be problematic, but usage on the consumer side is kept simple
  because of this

Watch this presentation by Runar Bjarnason on this subject
because it's awesome: [Constraints Liberate, Liberties Constrain](https://www.youtube.com/watch?v=GqmsQeSzMdw)

### 4.3. SHOULD NOT wrap purely CPU-bound operations in Scala's standard Futures

This is in general an anti-pattern:

```scala
def add(x: Int, y: Int) = Future { x + y }
```

If you don't see any kind of I/O in there, then that's a red
flag. Shoving stuff in Futures without thinking is not going to solve
your performance problems. Especially in the case of a web server, in
which the requests are already paralellized and the above would get
executed in response to requests, shoving purely CPU-bound in that
Future constructor will make your logic slower to execute, not faster.

Also, in case you want to initialize a `Future[T]` with a constant,
always use `Future.successful()`.

### 4.4. MUST use Scala's BlockContext on blocking I/O

This includes all blocking I/O, including SQL queries. Real sample:

```scala
Future {
  DB.withConnection { implicit connection =>
    val query = SQL("select * from bar")
    query()
  }
}
```

Blocking calls are error-prone because one has to be aware of exactly
what thread-pool gets affected and given the default configuration of
the backend app, this can lead to non-deterministic dead-locks. It's a
bug waiting to happen in production.

Here's a simplified example demonstrating the issue for didactic purposes:

```scala
implicit val ec = ExecutionContext
  .fromExecutor(Executors.newFixedThreadPool(1))

def addOne(x: Int) = Future(x + 1)

def multiply(x: Int, y: Int) = Future {
  val a = addOne(x)
  val b = addOne(y)
  val result = for (r1 <- a; r2 <- b) yield r1 * r2

  // this will dead-lock
  Await.result(result, Duration.Inf)
}
```

This sample is simplified to make the effect deterministic, but all
thread-pools configured with upper bounds will sooner or later be
affected by this.

Blocking calls have to be marked with a `blocking` call that signals
to the `BlockContext` a blocking operation. It's a very neat mechanism
in Scala that lets the `ExecutionContext` know that a blocking operation
happens, such that the `ExecutionContext` can decide what to do about
it, such as adding more threads to the thread-pool (which is what
Scala's ForkJoin thread-pool does).

All the code has to be reviewed and whenever a blocking call happens,
this is the fix:

```scala
import scala.concurrent.blocking
// ...
blocking {
  someBlockingCallHere()
}
```

NOTE: the `blocking` call also serves as documentation, even if the
underlying thread-pool doesn't support `BlockContext`, as things that
block are totally non-obvious.

### 4.5. SHOULD NOT block

Sometimes you have to block the thread underneath - unfortunately JDBC
doesn't have a non-blocking API. However, when you have a choice,
never, ever block. For example, don't do this:

```scala
def fetchSomething: Future[String] = ???

// later ...
val result = Await.result(fetchSomething, 3.seconds)
result.toUpperCase
```

Prefer keeping the context of that Future all the way:

```scala
def fetchSomething: Future[String] = ???

fetchSomething.map(_.toUpperCase)
```

Also checkout [Scala-Async](https://github.com/scala/async) to make
this easier.

### 4.6. SHOULD use a separate thread-pool for blocking I/O

Related to
[Rule 4.4](#44-must-use-scalas-blockcontext-on-blocking-io), if you're
doing a lot of blocking I/O (e.g. a lot of calls to JDBC), it's better
to create a second thread-pool / execution context and execute all
blocking calls on that, leaving the application's thread-pool to deal
with CPU-bound stuff.

So you could do initialize this second execution context like:

```scala
import java.util.concurrent.Executors

// ...
private val ioThreadPool = Executors.newCachedThreadPool(
  new ThreadFactory {
    private val counter = new AtomicLong(0L)

    def newThread(r: Runnable) = {
      val th = new Thread(r)
      th.setName("eon-io-thread-" +
      counter.getAndIncrement.toString)
      th.setDaemon(true)
      th
    }
  })
```

Note that here I prefer to use an unbounded "cached thread-pool", so
it doesn't have a limit. When doing blocking I/O the idea is that
you've got to have enough threads that you can block. But if unbounded
is too much, depending on use-case, you can later fine-tune it, the
idea with this sample being that you get the ball rolling.

And then you could provide a helper, like:

```scala
def executeBlockingIO[T](cb: => T): Future[T] = {
  val p = Promise[T]()

  ioThreadPool.execute(new Runnable {
    def run() = try {
      p.success(blocking(cb))
    }
    catch {
      case NonFatal(ex) =>
        logger.error(s"Uncaught I/O exception", ex)
        p.failure(ex)
    }
  })

  p.future
}
```

Also, don't expose this I/O thread-pool as an execution context that
can be imported as an implicit in scope, because then people would
start using it for CPU-bound stuff by mistake, so it's better to hide
it and provide this helper.

### 4.7. All public APIs SHOULD BE thread-safe

As a general rule of software engineering on top of the JVM,
absolutely all public APIs from inside your process will end up being
used in a context in which multiple threads are using these APIs at
the same time. It's best practice for all public APIs (all components
in your Cake for example) to be designed as thread-safe from the get
go, because you do not want to chase concurrency bugs.

And if a public API is not thread-safe for some reason (like the usual
compromises made in software development), then state this fact in
BOLD CAPITAL LETTERS.

The reason is simple - if an API is not thread-safe, then there's no
way a user of that API can know about the way to synchronize on
it. Example:

```scala
val list = mutable.List.empty[String]
```

Lets say this mutable list is exposed. On what lock is a user supposed
to synchronize? On the list itself? That's not enough, being a pretty
useless lock to have in a larger context. Remember, locks are not
composable and are very error-prone. Never leave the responsibility of
synchronizing for contention on your users.

### 4.8. SHOULD avoid contention on shared reads

Meet
[Amdahl's Law](https://en.wikipedia.org/wiki/Amdahl's_law). Synchronizing
with locks drastically limits the parallelization possible and thus
vertical scalability. Reads are embarrassingly paralellizable, so
avoid at all times doing this:

```scala
def fetch = synchronized { someValue }
```

Come up with better synchronization schemes that does not involve
synchronizing reads, like atomic references or STM. If you aren't able
to do that, then avoid this altogether by using proper abstractions.

### 4.9. MUST provide a clearly defined and documented protocol for each component or actor that communicates over async boundaries

A function signature is not enough for documenting the protocol of
problematic components. Especially when talking about communications
over asynchronous boundaries (between threads, between processes on
the network, etc...), the protocol needs a lot of detail on what you
can and cannot rely on.

As a guideline, don't shy away from writing comments and document:

- concurrency and latency concerns
- the proper ordering of calls
- everything that can go wrong

### 4.10. SHOULD always prefer single-producer scenarios

Shared writes are not parallelizable, whereas shared reads are
embarrassingly parallelizable. As a metaphor, 100,000 people can watch
the same soccer game on the same stadium at the same time (reads), but
100,000 people cannot all use the same bathroom (writes). In a
multi-threading scenario, prefer single producer / multi consumer
scenarios. This has the effect of avoiding contention and performance
problems. An app does not scale vertically with multiple producers
pounding on the same resource, because Amdahl's Law.

Checkout [LMAX Disruptor](https://lmax-exchange.github.io/disruptor/).

### 4.11. MUST NOT hardcode the thread-pool / execution context

This is a general design issue related to the project as a whole, but don't do this:

```scala
import scala.concurrent.ExecutionContext.Implicits.global

def doSomething: Future[String] = ???
```

Tight coupling between the execution context and your logic is not
good and that import is tight-coupling, especially since in the
context of a Play Framework application you need to use a
[different thread-pool](https://www.playframework.com/documentation/2.5.x/ThreadPools).

Just pass the ExecutionContext around as an implicit parameter. It's
idiomatic and acceptable this way. Also, implicit parameters should be passed in the second group of parameters to avoid confusing implicit resolution. When passed in the first group, it allows for method calls like ```doSomething()``` that won't compile but most IDEs will show them as valid.

```scala
def doSomething()(implicit ec: ExecutionContext): Future[String] = ???
```
