# [译] 认识 rxjs 中的 BehaviorSubject、ReplaySubject 以及 AsyncSubject

> 原文链接：[Understanding rxjs BehaviorSubject, ReplaySubject and AsyncSubject](https://medium.com/@luukgruijs/understanding-rxjs-behaviorsubject-replaysubject-and-asyncsubject-8cc061f1cfc0)<br/>
> 原文作者：[Luuk Gruijs](https://medium.com/@luukgruijs)；发表于2018年5月4日<br/>
> 译者：[yk](https://github.com/m8524769)；如需转载，请注明[出处](https://github.com/m8524769/RxJS-Article-Translation)，谢谢合作！

Subject 的作用是实现 Observable 的多播。由于其 Observable execution 是在多个订阅者之间共享的，所以它可以确保每个订阅者接收到的数据绝对相等。不仅使用 Subject 可以实现多播，RxJS 还提供了一些 Subject 的变体以应对不同场景，那就是：BehaviorSubject、ReplaySubject 以及 AsyncSubject。

如果你还不知道 Subject（主题）是什么的话，建议先读读我的上一篇文章：[认识 rxjs 中的 Subject](https://github.com/m8524769/RxJS-Article-Translation/blob/master/Understanding-rxjs-Subjects/README.md)。如果你觉得OK没问题，那咱就继续！

![](assets/1_3xWKyEoCO-6DY2DkQKFpvw.jpeg)

> 摄影：[Cory Schadt](https://unsplash.com/photos/wXGKpIIMXOA?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)，来自 [Unsplash](https://unsplash.com/search/photos/stream?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

## BehaviorSubject

BehaviorSubject 是 Subject 的变体之一。**BehaviorSubject 的特性就是它会存储“当前”的值。这意味着你始终可以直接拿到 BehaviorSubject 最后一次发出的值。**

有两种方法可以拿到 BehaviorSubject “当前”的值：访问其 `.value` 属性或者直接订阅。如果你选择了订阅，那么 BehaviorSubject 将直接给订阅者发送当前存储的值，无论这个值有多么“久远”。请看下面的例子：

```javascript
import * as Rx from "rxjs";

const subject = new Rx.BehaviorSubject(Math.random());

// 订阅者 A
subject.subscribe((data) => {
  console.log('Subscriber A:', data);
});

subject.next(Math.random());

// 订阅者 B
subject.subscribe((data) => {
  console.log('Subscriber B:', data);
});

subject.next(Math.random());

console.log(subject.value)

// 输出
// Subscriber A: 0.24957144215097515
// Subscriber A: 0.8751123892486292
// Subscriber B: 0.8751123892486292
// Subscriber A: 0.1901322109907977
// Subscriber B: 0.1901322109907977
// 0.1901322109907977
```

详细讲解一下：

1. 我们首先创建了一个 subject，并在其中注册了一个订阅者A。由于 subject 是一个 BehaviorSubject，所以订阅者A 会立即收到 subject 的初始值（一个随机数），并同时将其打印。
2. subject 随即广播下一个值。订阅者A 再次打印收到的值。
3. 第二次订阅 subject 并称其为订阅者B。同样，订阅者B 也会立即收到 subject 当前存储的值并打印。
4. subject 再次广播下一个新的值。这时，两个订阅者都会收到这个值并打印。
5. 最后，我们通过简单地访问 `.value` 属性的形式获取了 subject 当前的值并打印。这在同步的场景下非常好用，因为你不必订阅 Subject 就可以获取它的值。

另外，你可能发现了 BehaviorSubject 在创建时是需要设置一个初始值的。这一点在 Observable 上就非常难实现，而在 BehaviorSubject 上，只要传递一个值就行了。

> 译者注：当前版本的 RxJS 中 `BehaviorSubject()` 是必须要设置初始值的，否则会导致执行错误，而原文并没有体现这一点。所以我在这一段做了较多改动，以免误导读者。详见 [BehaviorSubject](https://rxjs-dev.firebaseapp.com/api/index/class/BehaviorSubject)。

## ReplaySubject

相比 BehaviorSubject 而言，ReplaySubject 是可以给新订阅者发送“旧”数据的。另外，ReplaySubject 还有一个额外的特性就是它可以记录一部分的 observable execution，从而存储一些旧的数据用来“重播”给新来的订阅者。

当创建 ReplaySubject 时，你可以指定存储的数据量以及数据的过期时间。也就是说，你可以实现：给新来的订阅者“重播”订阅前一秒内的最后五个已广播的值。示例代码如下：

```javascript
import * as Rx from "rxjs";

const subject = new Rx.ReplaySubject(2);

// 订阅者 A
subject.subscribe((data) => {
  console.log('Subscriber A:', data);
});

subject.next(Math.random())
subject.next(Math.random())
subject.next(Math.random())

// 订阅者 B
subject.subscribe((data) => {
  console.log('Subscriber B:', data);
});

subject.next(Math.random());

// Subscriber A: 0.3541746356538569
// Subscriber A: 0.12137498878080955
// Subscriber A: 0.531935186034298
// Subscriber B: 0.12137498878080955
// Subscriber B: 0.531935186034298
// Subscriber A: 0.6664809293975393
// Subscriber B: 0.6664809293975393
```

简单解读一下代码：

1. 我们创建了一个 ReplaySubject 并指定其只存储最近两次广播的值；
2. 订阅 subject 并称其为订阅者A；
3. subject 连续广播三次，同时订阅者A 也会跟着连续打印三次；
4. 这一步就轮到 ReplaySubject 展现魔力了。我们再次订阅 subject 并称其为订阅者B，因为之前我们指定 subject 存储最近两次广播的值，所以 subject 会将上两个值“重播”给订阅者B。我们可以看到订阅者B 随即打印了这两个值；
5. subject 最后一次广播，两个订阅者收到值并打印。

之前提到了你还可以设置 ReplaySubject 的数据过期时间。让我们来看看下面这个例子：

```javascript
import * as Rx from "rxjs";

const subject = new Rx.ReplaySubject(2, 100);

// 订阅者A
subject.subscribe((data) => {
  console.log('Subscriber A:', data);
});

setInterval(() => subject.next(Math.random()), 200);

// 订阅者B
setTimeout(() => {
  subject.subscribe((data) => {
    console.log('Subscriber B:', data);
  });
}, 1000)

// Subscriber A: 0.44524184251927656
// Subscriber A: 0.5802631630066313
// Subscriber A: 0.9792165506699135
// Subscriber A: 0.3239616040117268
// Subscriber A: 0.6845077617520203
// Subscriber B: 0.6845077617520203
// Subscriber A: 0.41269171141525707
// Subscriber B: 0.41269171141525707
// Subscriber A: 0.8211466186035139
// Subscriber B: 0.8211466186035139
```

同样解读一下代码：

1. 我们创建了一个 ReplaySubject，指定其存储最近两次广播的值，但只保留 100ms；
2. 订阅 subject 并称其为订阅者A；
3. 我们让这个 subject 每 200ms 广播一次。订阅者A 每次都会收到值并打印；
4. 我们设置在程序执行一秒后再次订阅 subject，并称其为订阅者B。这意味着在它开始订阅之前，subject 就已经广播过五次了。由于我们在创建 subject 时就设置了数据的过期时间为 100ms，而广播间隔为 200ms，所以订阅者B 在开始订阅后只会收到前五次广播中最后一次的值。

## AsyncSubject

BehaviorSubject 和 ReplaySubject 都可以用来存储一些数据，而 AsyncSubject 就不一样了。AsyncSubject 只会在 Observable execution 完成后，将其最终值发给订阅者。请看代码：

```javascript
import * as Rx from "rxjs";

const subject = new Rx.AsyncSubject();

// 订阅者A
subject.subscribe((data) => {
  console.log('Subscriber A:', data);
});

subject.next(Math.random())
subject.next(Math.random())
subject.next(Math.random())

// 订阅者B
subject.subscribe((data) => {
  console.log('Subscriber B:', data);
});

subject.next(Math.random());
subject.complete();

// Subscriber A: 0.4447275989704571
// Subscriber B: 0.4447275989704571
```

虽然这段代码的输出并不多，但我们还是照例解读一下：

1. 创建 AsyncSubject；
2. 订阅 subject 并称其为订阅者A；
3. subject 连续广播三次，但什么也没发生；
4. 再次订阅 subject 并称其为订阅者B；
5. subject 又广播了一次，但还是什么也没发生；
6. subject 完成。两个订阅者这才收到传来的值并打印至终端。

## 结论

BehaviorSubject、ReplaySubject 和 AsyncSubject 都可以像 Subject 一样被用于多播。它们各自都有一些额外的特性以适用于不同场景。
