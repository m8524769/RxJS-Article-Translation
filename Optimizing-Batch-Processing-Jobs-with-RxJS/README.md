# [译] 使用 RxJS 优化批处理作业

> 原文链接：[Optimizing Batch Processing Jobs with RxJS](https://medium.com/@ravishivt/batch-processing-with-rxjs-6408b0761f39)<br/>
> 原文作者：[Ravi Mehra](https://medium.com/@ravishivt)；发表于2019年4月8日<br/>
> 译者：[yk](https://github.com/m8524769)；如需转载，请注明[出处](https://github.com/m8524769/RxJS-Article-Translation)，谢谢合作！

批处理并不是什么稀奇的东西。不过是检索数据，处理数据，周而复始。它没有花哨的界面，只是接受尽可能少的输入，然后默默地完成工作。但探索如何优化批处理却是一个非常有意思的过程。当你一点一点地提升代码的效率时，当你将程序运行时间从几天缩短至几小时，甚至几分钟时，你会觉得自己简直就是一个算法天才。

优化策略通常是趋同的：在避免资源过载的情况下，最大限度地提升并发性能。在两者之间做出权衡会是一大挑战。因此，我们需要正确的工具集和处理模型来达到优化目的。[RxJS](https://rxjs.dev/guide/overview) 正是其中之一。

在本文中，我们将着眼于使用 RxJS 以响应式的方法来处理常见的批处理模式。我们还会将此与传统的方式进行比较，并基于视图和基准进行评估。虽然响应式的方法会使编码过程更为复杂，但当你在设计下一个批处理作业时，性能和灵活性的提升会使 RxJS 变得尤为瞩目。

下图体现了各个方案之间效率的差距，最终我们将达到 5.3 倍的提速！

![](assets/1_MrAFZusUC1UO4f3g00GyqQ.png)

## 概要

Observable（可观察对象）是个非常棒的东西，可用于提高批处理作业的效率，但代价是我们需要额外地以响应式的思维方式来思考问题，也许最终并不会提升多少性能，徒增复杂度。

如果你已经很熟悉 RxJS 了，那么我建议你倒过来读这篇文章。首先阅读并分析[使用 observable 的最佳优化代码](https://codesandbox.io/s/m5qmvnklkj?module=%2Fsrc%2Fapproaches%2F4-observableImproved.ts)以获得粗略理解。然后再回到文章中，搞明白它的工作机制以及为何比使用 `Promise.all()` 要来的好。如果你并不是很熟悉 RxJS，也可以这么干，但是会比较吃力。

## 批处理的背景

以下是一些批处理作业的例子：

- 遍历 API 的分页数据列表，获取每条记录的详细信息，并在 ElasticSearch 中对其进行索引。
- 从数据库中获取数据，与其他数据进行聚合，生成 HTML 报告并通过电子邮件的形式发送出去。
- 读取数据库并为网络爬虫生成 sitemap.xml。
- 使用种子数据（seed data）初始化数据库，可以是随机数据。

> 译者注：[Elasticsearch](https://www.elastic.co/cn/) 是一个开源的分布式 RESTful 搜索和分析引擎。

批处理作业可以通过两种方式启动：由开发人员手动执行或按计划任务自动执行（task runner）。理想情况下，手动执行的作业需要几秒钟，或最多几分钟来完成。然而对于开发人员来说，每一秒都很重要。因为在执行完成之前，我们只能干等着。对于我而言，在终端前等的时间越长，就越有可能分心。注意力涣散，意识模糊，然后就刷起了知乎。理由很简单：“与其干等几分钟，不如刷会儿知乎放松放松”。然后刷着刷着，十几分钟就过去了。

对于自动执行的计划任务来说，耗时较长的任务就不那么碍眼了。这种任务一般只要设置完就可以把它抛到脑后了。然而，其糟糕的性能意味着我们不能频繁执行它。对于有些耗时超过一小时的任务，我们就会把它设置成每两小时执行一次，而非每小时都执行。在任务执行期间，其处理的数据集也可能会因为时间的推移而变“脏”。如果一个计划任务的工作是每小时收集一次统计数据，却要耗时 40 分钟，那么报告中的数据究竟属于哪个时间点？

不过好在几乎所有的批处理作业都可以通过并发（concurrency）来提高效率，而且效果十分明显。由于批处理作业会遍历数据集并执行某些操作，因此采用并发就能显著提高效率。但是，以高效的方式实现并发处理并没有那么容易。

在 JavaScript 中，有很多工具可以帮助我们利用异步操作的并发性，比如 Promise、ES2017 中的 async/await、ES2018 中的异步迭代（async iteration）、NodeJS 流，以及一些第三方库，比如 [bluebird](http://bluebirdjs.com/docs/getting-started.html)、[scramjet](https://github.com/signicode/scramjet) 和 [async](https://github.com/caolan/async)。还有一个就是在前端应用中较为流行的：[RxJS](https://github.com/ReactiveX/rxjs)。这些工具各有千秋，我们会将 RxJS 与其余的几个进行比较。

## RxJS

[RxJS](https://github.com/ReactiveX/rxjs) 在前端开发方面有着很大的吸引力，因为它与前端环境是完美适配的。它将用户视为与应用程序交互并随时会发出值的 Observable。按钮点击、页面导航、文本输入，所有的这类事件都会被应用程序以结构化的方式观察到。事件会通过代码传播，从而导致状态变更，触发网络请求或者 DOM 的更改。RxJS 提供了大量的操作符，你可以通过你想要的任何方式来处理事件。

鲜为人知的是，RxJS 也是批处理的绝佳工具。在批处理场景下，我们可以将数据源（如：数据库）视为一个 Observable，随着时间的推移发出我们所需的批量数据。然后我们可以使用诸如 mergeMap 或 concatMap 之类的 Observable 操作符来处理数据。

本文不会过多介绍 RxJS/Observable 以及它们的工作方式。因为涉及的面太广，而且已经太多优秀的文章可以让我们学习了。我们将探索如何使用 RxJS 来高效的解决批处理作业，并仔细分析其中的原理。如果你是 RxJS 的新手，我建议你通过这两篇文章来获得对 RxJS 的初步了解：[The introduction to Reactive Programming you’ve been missing](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754) 和 [Learn to combine RxJs sequences with super intuitive interactive diagrams](https://blog.angularindepth.com/learn-to-combine-rxjs-sequences-with-super-intuitive-interactive-diagrams-20fce8e6511)

> 译者注：这两篇文章都已有对应译文，分别是[极客学院 Wiki](http://wiki.jikexueyuan.com/) 的[响应式编程（Reactive Programming）介绍](http://wiki.jikexueyuan.com/project/android-weekly/issue-145/introduction-to-RP.html)和 [RxJS 中文社区](https://github.com/RxJS-CN)的 [ [译] RxJS: 使用超直观的交互图来学习组合操作符](https://github.com/RxJS-CN/rxjs-articles-translation/blob/master/articles/Learn-To-Combine-RxJS-Sequences-With-Super-Intuitive-Interactive-Diagrams.md)

注意，并不是说只要将批处理作业转换为 RxJS 和 Observable 就可以提高效率的。我们需要设计操作符之间的链式调用来使整体的效率最大化。例如，我们可以利用 `mergeMap` 操作符和 `concurrent` 选项来同时执行一部分操作。

### 背压（Back-pressure）

_本节更适合有 RxJS 使用经验的读者，初学者可随意跳过。_

我认为 RxJS 没有被广泛地用于批处理有这样两个原因：其一，额外的学习成本；其二，缺乏处理无损背压的案例。背压是一个比较复杂的话题，其核心问题是：生产者提供数据的速度快于消费者可以接收并处理数据的速度。你可以想象一个排水管被塞住一半的水槽，由于其排水（处理数据）的过程会比平常来的更慢，所以水槽（堆内存）会被逐渐占满并最终溢出。

批处理作业通常需要以受控方式来处理海量数据（分批/分页/分块）。如果不这样做，那么程序就会因以下原因而崩溃：

- 内存被挤爆；
- 数据库连接耗尽；
- 达到 API 服务器的访问频率限制；
- API/数据库性能骤降。

因此，背压管理的重要性不言而喻。RxJS 提供了有损和无损两种背压策略，但对于批处理作业而言，有损是不用考虑的，因为我们不能跳过任何数据。

我发现 RxJS 中的几乎所有无损背压方案都需要在声明 observable 时提供整个数据集，例如：`from(entireDatasetArray)`，而这是不现实的。所以，我们需要一种“响应式拉取（reactive pull）”的方式来替代，也就是控制数据拉取的频率。

在本文提供的示例中，我将借助 RxJS 的 `Subject` 来实现一种更为清晰的的背压管理解决方案。简而言之，就是在处理了一批数据之后，调用 `Subject.next()` 来拉取/获取下一批数据。这样，我们就可以随时控制数据流的大小。以下两篇文章是我的灵感来源：

- [RXJS control observable invocation](https://stackoverflow.com/a/35347136/684893)
- [Lossless Backpressure in RxJS](https://itnext.io/lossless-backpressure-in-rxjs-b6de30a1b6d4)

## 示例——场景及设定

我们需要一个相对真实的批处理场景来评估现有的解决方案。为了使其有趣一些，我会施加一些限制。下面就是我们将要处理的场景：

1. 以分页的方式从数据库中获取公司列表（每页 10 条）。每次获取的记录条数应当为 `batchSize` 的值，并且内存每次读取的记录条数不应超过 `maxQueueSize`。另外，同时执行的数据库查询语句条数也不应当超过 `retrieveCompaniesConcurrency`。
2. 获取每个公司的订单列表。假设其对应了一个 API 请求，为了避免频繁请求而触发访问限制，我们会设置 `retrieveOrdersConcurrency` 参数来控制请求的并发量。
3. 将收集到的各公司订单信息做成报告，并通过电子邮件的方式发给他们。假设我们有一个处理电子邮件的 API 服务器，它可以单独地收发邮件，但在同时收发多个邮件的情况下性能会更好。我们应当将单个 API 请求中的邮件数量限制为 `bulkEmailCount`。同时，也要像获取订单一样设置一个 `bulkEmailConcurrency` 来限制访问的频率。

为了演示该场景，我们将模拟上述三个异步操作。

1. `retrieveCompanies(limit: number, offset: number): Promise<Company[]>` 使用 [faker.js](https://github.com/marak/Faker.js/) 来生成公司数据，数据量为 `totalCompanyCount`，支持 `limit` 和 `offset` 参数来获取分页数据。
2. `retrieveCompanyOrders(company: Company): Promise<Order[]>` 再次使用 faker.js 为每个公司生成个数为 `ORDERS_PER_COMPANY` 的订单。
3. `sendBulkEmails(emailData: Company[]): Promise<void>` 一个无操作（no-op）调用，因为我们不期望得到任何回复。

我们将用 `setTimeout()` 为每个操作设置一个延时，从而模拟一个真实的异步调用环境，且所有的延时都是可调整的参数。出于本文的目的，我们将使用以下这些常量来设定参数，以便公平地比较各个解决方案。

```typescript
export const defaultBatchProcessingOptions: BatchProcessingOptions = {
  batchSize: 5,
  maxQueueSize: 15,
  retrieveCompaniesConcurrency: 1,
  retrieveOrdersConcurrency: 5,
  bulkEmailConcurrency: 5,
  maxBulkEmailCount: 5,
};
const TOTAL_COMPANY_COUNT = 100;
const RETRIEVE_ONE_COMPANY_DELAY = 6;
const RETRIEVE_ONE_COMPANY_ORDER_DELAY = 5;
const ORDERS_PER_COMPANY = 6;
const SEND_BULK_EMAILS_DELAY = 60;
```

在设置了这些参数后，我们可以计算出：

- `retrieveCompanies()` 将耗时 30ms（`batchSize * RETRIEVE_ONE_COMPANY_DELAY === 30`）
- `retrieveCompanyOrders()` 也将耗时 30ms（`ORDERS_PER_COMPANY * RETRIEVE_ONE_COMPANY_DELAY === 30`）
- `sendBulkEmails()` 将耗时 60ms（`SEND_BULK_EMAILS_DELAY`）。

观察延时的调整以及随机因素的添加将如何影响各个方法的运行时间是个十分有趣的过程，其结果并非总是呈线性的。本文将尝试各种各样的参数配置，你可以随意更改这些参数。正如我们稍后将看到的，优化后的 observable 解决方案实际上可以利用在其他方案中难以解决的随机性。

并非所有方案都要用到这些参数，有些参数仅在一些进阶场景下才用得到。对于简单的场景，我们不会违反任何约束条件，因为那样效率太低了。

在 [_`utils.ts`_](https://codesandbox.io/s/m5qmvnklkj?fontsize=14&module=%2Fsrc%2Futils.ts&view=editor) 中，你可以找到上述异步操作的完整实现，包括生成随机数据以及批处理的参数选项。

<iframe src="https://codesandbox.io/embed/m5qmvnklkj?fontsize=14&module=%2Fsrc%2Futils.ts&view=editor" title="rxjs-batch-processing" allow="geolocation; microphone; camera; midi; vr; accelerometer; gyroscope; payment; ambient-light-sensor; encrypted-media" style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;" sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"></iframe>

另外，[_`index.ts`_](https://codesandbox.io/s/m5qmvnklkj?fontsize=14&module=%2Fsrc%2Findex.ts&view=editor) 文件是我们测试每一种方案的入口。每种方案都会测试三次，并最终输出平均结果。如果你对某些测试不感兴趣的话，可以在下面评论。

<iframe src="https://codesandbox.io/embed/m5qmvnklkj?fontsize=14&module=%2Fsrc%2Findex.ts&view=editor" title="rxjs-batch-processing" allow="geolocation; microphone; camera; midi; vr; accelerometer; gyroscope; payment; ambient-light-sensor; encrypted-media" style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;" sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"></iframe>
