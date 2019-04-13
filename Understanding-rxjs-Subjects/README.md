# [译] 认识 rxjs 中的 Subjects

> 原文链接：[Understanding rxjs Subjects](https://medium.com/@luukgruijs/understanding-rxjs-subjects-339428a1815b)<br/>
> 原文作者：[Luuk Gruijs](https://medium.com/@luukgruijs)；发表于2018年4月17日<br/>
> 译者：[yk](https://github.com/m8524769)；如需转载，请注明[出处](https://github.com/m8524769/RxJS-Article-Translation)，谢谢合作！

RxJS 是真的好用，它可以帮助你编辑/订阅数据流。也许你用一些“简单”的 Observables（可观察对象）就可以做很多事情。但你要知道 RxJS 还是提供了非常多的类用来操控数据流，Subject（主题）就是其中之一。

如果你还不知道 Observable 是什么的话，建议先读读我的另一篇文章：[认识，创建并订阅 Angular 中的可观察对象](https://medium.com/@luukgruijs/understanding-creating-and-subscribing-to-observables-in-angular-426dbf0b04a3)。如果你觉得你明白 Observable 是什么意思，那没问题咱们继续！

![](assets/1_Aovk5ccg_BORDRKkVL9h0A.jpeg)

> 摄影：[Anvesh Uppunuthula](https://unsplash.com/photos/GlT_pCXNU4o?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)，来自 [Unsplash](https://unsplash.com/search/photos/stream?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

## Subjects

Subject 就好比一个 Observable。它可以被订阅，就像你平时订阅 Observable 一样。它也有类似 `next()`，`error()` 以及 `complete()` 的方法，就像你平时传给 Observable 构造函数的 observer（观察者）一样。

使用 Subjects 主要是为了多播（multicast）。Observable 默认是单播（unicast）的。而单播就意味着：对于每个订阅的观察者，都只有一个独立的 Observable execution。为了证明这一点：

```javascript
import * as Rx from "rxjs";

const observable = Rx.Observable.create((observer) => {
  observer.next(Math.random());
});

// 订阅 1
observable.subscribe((data) => {
  console.log(data); // 0.24957144215097515 (随机数)
});

// 订阅 2
observable.subscribe((data) => {
  console.log(data); // 0.004617340049055896 (随机数)
});
```

由于 Observable 在设计上就是单播的，所以如果你希望使多个订阅者收到相同的数据，那可能会非常麻烦。而 Subjects 可以帮助我们解决这个问题。正如先前所说，Subjects 可以用来实现多播。多播的基本含义是：一个 Observable execution 可以在多个订阅者之间共享。

> 译者注：当我们调用 `Observable.subscribe()` 时，一个 _Observable execution_ 就会被启动，详见 [Observable](https://rxjs-dev.firebaseapp.com/api/index/class/Observable#subscribe)。

**Subjects 就像事件发射器（EventEmitters），管理着多个已注册的事件监听器。** 当我们订阅一个 Subject 时，它并不会启动一个新的 execution 来传送数据。而是在现有观察者列表中注册一个新的观察者，仅此而已。

## 如何在多播中使用 Subjects

多播是 Subject 的一个特性，你无需使用任何技巧即可实现这一特性。下面是一个简单的示例：

```javascript
import * as Rx from "rxjs";

const subject = new Rx.Subject();

// 订阅者 1
subject.subscribe((data) => {
  console.log(data); // 0.24957144215097515 (随机数)
});

// 订阅者 2
subject.subscribe((data) => {
  console.log(data); // 0.24957144215097515 (随机数)
});

subject.next(Math.random());
```

奈斯！我们使两次订阅获得了相同的数据。然而，这并非 Subjects 的唯一用途。

相比 Observables 只能作为数据的生产者，Subjects 即可以作为生产者，也可以作为消费者。通过把 Subjects 作为消费者使用，你可以将一个单播 Observable 转换为多播。示例如下：

```javascript
import * as Rx from "rxjs";

const observable = Rx.Observable.create((observer) => {
  observer.next(Math.random());
});

const subject = new Rx.Subject();

// 订阅者 1
subject.subscribe((data) => {
  console.log(data); // 0.24957144215097515 (随机数)
});

// 订阅者 2
subject.subscribe((data) => {
  console.log(data); // 0.24957144215097515 (随机数)
});

observable.subscribe(subject);
```

将我们的 Subject 传递给 `subscribe()`，使其接收由 Observable 传来的值（消费数据）。随后，Subject 的所有订阅者都会立即收到这个值。

## 结论

当你发现 Observable 的多个订阅接收到的数据不同时，就可以使用 Subjects 来解决。由于 Observable execution 可以在多个订阅者之间共享，所以 Subjects 将确保每个订阅接收到的数据是绝对相等的。另外，Subjects 还有几种变体，我将在下一篇文章中详细描述它们之间的区别。

## 想在阿姆斯特丹找到一份工作吗？

> 译者注：后面都是一些无关紧要的招聘信息，我就不翻译了，除非你真的想去阿姆斯特丹。
