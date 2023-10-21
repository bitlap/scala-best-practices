# Scala Best Practices

<img src="./assets/scala-logo-256.png"  align="right" width="100" height="100" />

- 版本： `1.3` 
- 更新时间: `2023-10-20`
- 翻译&维护者：[梦境迷离](https://github.com/bitlap/scala-best-practices)

## Table of Contents

- [0. 序言](sections/0-preface.md)
  - [0.1 切勿盲从建议](sections/0-preface.md#01-must-not-follow-advice-blindly)

- [1. 卫生规范](sections/1-hygienic-rules.md)
  - [1.1. SHOULD 强制执行合理的代码行长度](sections/1-hygienic-rules.md#11-should-enforce-a-reasonable-line-length)
  - [1.2. MUST NOT 依靠 SBT 或 IDE 插件为您进行格式化](sections/1-hygienic-rules.md#12-must-not-rely-on-a-sbt-or-ide-plugin-to-do-the-formatting-for-you)
  - [1.3. SHOULD 打破长函数](sections/1-hygienic-rules.md#13-should-break-long-functions)
  - [1.4. MUST NOT 在命名和注释中引入拼写错误](sections/1-hygienic-rules.md#14-must-not-introduce-spelling-errors-in-names-and-comments)
  - [1.5. MUST 有意义的命名](sections/1-hygienic-rules.md#15-names-must-be-meaningful)

- [2. 语言规范](sections/2-language-rules.md)
  - [2.1. MUST NOT 使用`return`](sections/2-language-rules.md#21-must-not-use-return)
  - [2.2. SHOULD 使用不可变数据结构](sections/2-language-rules.md#22-should-use-immutable-data-structures)
  - [2.3. SHOULD NOT 使用循环或条件更新一个`var`](sections/2-language-rules.md#23-should-not-update-a-var-using-loops-or-conditions)
  - [2.4. SHOULD NOT 定义无用的特质](sections/2-language-rules.md#24-should-not-define-useless-traits)
  - [2.5. MUST NOT 在样例类中使用`var`](sections/2-language-rules.md#25-must-not-use-var-inside-a-case-class)
  - [2.6. SHOULD NOT 声明抽象的`var`成员](sections/2-language-rules.md#26-should-not-declare-abstract-var-members)
  - [2.7. MUST NOT 在验证用户输入或流控制时抛出异常](sections/2-language-rules.md#27-must-not-throw-exceptions-for-validations-of-user-input-or-flow-control)
  - [2.8. MUST NOT 捕获`Throwable`](sections/2-language-rules.md#28-must-not-catch-throwable-when-catching-exceptions)
  - [2.9. MUST NOT 使用`null`](sections/2-language-rules.md#29-must-not-use-null)
  - [2.10. MUST NOT 使用`Option.get`](sections/2-language-rules.md#210-must-not-use-optionget)
  - [2.11. MUST NOT 使用Java的日期或日历，而不是使用`java.time`(JSR-310)](sections/2-language-rules.md#211-must-not-use-javas-date-or-calendar-instead-use-javatime-jsr-310)
  - [2.12. SHOULD NOT 使用`Any`或`AnyRef`或`isInstanceOf`/`asInstanceOf`](sections/2-language-rules.md#212-should-not-use-any-or-anyref-or-isinstanceof--asinstanceof)
  - [2.13. MUST 将日期序列化为 Unix 时间戳或 ISO 8601](sections/2-language-rules.md#213-must-serialize-dates-as-either-unix-timestamp-or-as-iso-8601)
  - [2.14. MUST NOT 使用魔法值](sections/2-language-rules.md#214-must-not-use-magic-values)
  - [2.15. SHOULD NOT 使用`var`作为共享状态](sections/2-language-rules.md#215-should-not-use-var-as-shared-state)
  - [2.16. SHOULD 公共函数应该具有显式的返回类型](sections/2-language-rules.md#216-public-functions-should-have-an-explicit-return-type)
  - [2.17. SHOULD NOT 定义嵌套在其他类中的样例类](sections/2-language-rules.md#217-should-not-define-case-classes-nested-in-other-classes)
  - [2.18. MUST NOT 在包对象中包含类、特质和对象](sections/2-language-rules.md#218-must-not-include-classes-traits-and-objects-inside-package-objects)
  - [2.19. SHOULD 只有当`head`/`tail`和`init`/`last`分解可以在常量时间和内存中完成时，才使用它们](sections/2-language-rules.md#219-should-use-headtail-and-initlast-decomposition-only-if-they-can-be-done-in-constant-time-and-memory)
  - [2.20. MUST NOT 使用`Seq.head`](sections/2-language-rules.md#220-must-not-use-seqhead)
  - [2.21. SHOULD 样例类标记为`final`](sections/2-language-rules.md#221-case-classes-should-be-final)
  - [2.22. SHOULD NOT 使用`scala.App`](sections/2-language-rules.md#222-should-not-use-scalaapp)

- [3. 应用架构](sections/3-architecture.md)
  - [3.1. SHOULD NOT 使用 Cake pattern](sections/3-architecture.md#31-should-not-use-the-cake-pattern)
  - [3.2. MUST NOT 将内容放入 Play 的 Global](sections/3-architecture.md#32-must-not-put-things-in-plays-global)
  - [3.3. SHOULD NOT 应用优化而不进行分析](sections/3-architecture.md#33-should-not-apply-optimizations-without-profiling)
  - [3.4. SHOULD 注意垃圾收集器](sections/3-architecture.md#34-should-be-mindful-of-the-garbage-collector)
  - [3.5. MUST NOT 使用无参数`ConfigFactory.load()`或直接访问`Config`对象](sections/3-architecture.md#35-must-not-use-parameterless-configfactoryload-or-access-a-config-object-directly)

- [4. 并发和并行](sections/4-concurrency-parallelism.md)
  - [4.1. SHOULD 视并发如瘟疫](sections/4-concurrency-parallelism.md#41-should-avoid-concurrency-like-the-plague-it-is)
  - [4.2. SHOULD 只在适当的情况下使用适当的抽象](sections/4-concurrency-parallelism.md#42-should-use-appropriate-abstractions-only-where-suitable---future-actors-rx)
  - [4.3. SHOULD NOT 在`Futures`中封装纯 CPU 绑定的操作](sections/4-concurrency-parallelism.md#43-should-not-wrap-purely-cpu-bound-operations-in-futures)
  - [4.4. MUST 在阻塞 I/O 时使用 Scala 的`BlockContext`](sections/4-concurrency-parallelism.md#44-must-use-scalas-blockcontext-on-blocking-io)
  - [4.5. SHOULD NOT 阻塞](sections/4-concurrency-parallelism.md#45-should-not-block)
  - [4.6. SHOULD 为阻塞 I/O 使用单独的线程池](sections/4-concurrency-parallelism.md#46-should-use-a-separate-thread-pool-for-blocking-io)
  - [4.7. SHOULD 所有公共 APIs 都应线程安全](sections/4-concurrency-parallelism.md#47-all-public-apis-should-be-thread-safe)
  - [4.8. SHOULD 避免共享读取时出现竞争s](sections/4-concurrency-parallelism.md#48-should-avoid-contention-on-shared-reads)
  - [4.9. MUST 为通过异步边界进行通信的每个组件或行为体提供定义明确并记录在案的协议](sections/4-concurrency-parallelism.md#49-must-provide-a-clearly-defined-and-documented-protocol-for-each-component-or-actor-that-communicates-over-async-boundaries)
  - [4.10. SHOULD 始终倾向于单个生产者的方案](sections/4-concurrency-parallelism.md#410-should-always-prefer-single-producer-scenarios)
  - [4.11. MUST NOT 硬编码线程池/执行上下文](sections/4-concurrency-parallelism.md#411-must-not-hardcode-the-thread-pool--execution-context)

- [5. Akka Actors](sections/5-actors.md)
  - [5.1. SHOULD 只在收到外部信息时才改变 actors 的状态](sections/5-actors.md#51-should-evolve-the-state-of-actors-only-in-response-to-messages-received-from-the-outside)
  - [5.2. SHOULD 只使用`context.been`来改变 actors 的状态](sections/5-actors.md#52-should-mutate-state-in-actors-only-with-contextbecome)
  - [5.3. MUST NOT 在异步闭包中泄漏actors的内部状态](sections/5-actors.md#53-must-not-leak-the-internal-state-of-an-actor-in-asynchronous-closures)
  - [5.4. SHOULD 使用背压](sections/5-actors.md#54-should-do-back-pressure)
  - [5.5. SHOULD NOT 使用 Akka FSM](sections/5-actors.md#55-should-not-use-akka-fsm)

---

## 贡献

打开问题以提出建议，或创建拉取请求 ;-)

---

Copyright &copy; 2015-2016, Some Rights Reserved.<br />
Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
