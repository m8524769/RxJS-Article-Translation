# [译] RxJS: 避免 takeUntil 造成的泄露风险

> 原文链接：[RxJS: Avoiding takeUntil Leaks](https://blog.angularindepth.com/rxjs-avoiding-takeuntil-leaks-fb5182d047ef)<br/>
> 原文作者：[Nicholas Jamieson](https://blog.angularindepth.com/@cartant)；发表于2018年5月27日<br/>
> 译者：[yk](https://github.com/m8524769)；如需转载，请注明[出处](https://github.com/m8524769/RxJS-Article-Translation)，谢谢合作！

![](assets/1_39OlUGRh9dUvY0_SE7ytHw.jpeg)

> 摄影：[Tim Gouw](https://unsplash.com/photos/_U-x3_FYxfI?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)，来自 [Unsplash](https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

使用 `takeUntil` 操作符来实现 observable 的自动取消订阅是 Ben Lesh 在 [Don’t Unsubscribe](https://medium.com/@benlesh/rxjs-dont-unsubscribe-6753ed4fda87) 中所提出的一种机制。

> 译者注：《Don’t Unsubscribe》在 [RxJS 中文社区](https://github.com/RxJS-CN)中已有相应[译文](https://github.com/RxJS-CN/rxjs-articles-translation/blob/master/articles/Don't-Unsubscribe.md)，有兴趣可以看看。

而该机制也是 Angular 中组件销毁所采用的[取消订阅模式](https://stackoverflow.com/a/41177163/6680611)的基础。

为了使该机制生效，我们必须以特定的顺序调用操作符。而我最近却发现有些人在使用 `takeUntil` 时，会因为操作符调用顺序的问题而导致订阅泄露。

让我们来看看哪些调用顺序是有问题的，以及导致泄露的原因。

## 哪些调用顺序是有问题的？

如果在 `takeUntil` 后面调用这样一个操作符，该操作符订阅了另一个 observable，那么当 `takeUntil` 收到通知时，该订阅可能不会被取消。

举个例子，这里的 `combineLatest` 可能会泄露 `b` 的订阅。

```typescript
import { Observable } from "rxjs";
import { combineLatest, takeUntil } from "rxjs/operators";

declare const a: Observable<number>;
declare const b: Observable<number>;
declare const notifier: Observable<any>;

const c = a.pipe(
  takeUntil(notifier),
  combineLatest(b)
).subscribe(value => console.log(value));
```

同样，这里的 `switchMap` 也可能会泄露 `b` 的订阅。

```typescript
import { Observable } from "rxjs";
import { switchMap, takeUntil } from "rxjs/operators";

declare const a: Observable<number>;
declare const b: Observable<number>;
declare const notifier: Observable<any>;

const c = a.pipe(
  takeUntil(notifier),
  switchMap(_ => b)
).subscribe(value => console.log(value));
```

## 为什么会导致泄露？

当我们用 `combineLatest` 静态工厂函数来代替已废弃的 `combineLatest` 操作符时，泄露的原因会更加明显，请看代码：

```typescript
import { combineLatest, Observable } from "rxjs";
import { takeUntil } from "rxjs/operators";

declare const a: Observable<number>;
declare const b: Observable<number>;
declare const notifier: Observable<any>;

const c = a.pipe(
  takeUntil(notifier),
  o => combineLatest(o, b)
).subscribe(value => console.log(value));
```

当 `notifier` 发出时，由 `takeUntil` 操作符返回的 observable 就算完成了，其订阅也会被自动取消。

然而，由于 `c` 的订阅者所订阅的 observable 并非由 `takeUntil` 返回，而是由 `combineLatest ` 返回，所以当 `takeUntil` 的 observable 完成时，`c` 的订阅是不会被自动取消的。

在 `combinedLast` 的所有 observable 全部完成之前，`c` 的订阅者都将始终保持订阅。所以，除非 `b` 率先完成，否则它的订阅就会被泄露。

**要想避免这个问题，我们就得把 `takeUntil` 放到最后调用：**

```typescript
import { combineLatest, Observable } from "rxjs";
import { takeUntil } from "rxjs/operators";

declare const a: Observable<number>;
declare const b: Observable<number>;
declare const notifier: Observable<any>;

const c = a.pipe(
  o => combineLatest(o, b),
  takeUntil(notifier)
).subscribe(value => console.log(value));
```

如此一来，当 `notifier` 发出时，`c` 的订阅就会被自动取消。因为当 `takeUntil` 中的 observable 完成时，`takeUntil` 会取消 `combineLatest ` 的订阅，这样也就依次取消了 `a` 和 `b` 的订阅。

## 使用 TSLint 来避免这个问题

如果你正在使用 `takeUntil` 的机制来实现间接地取消订阅，那么你可以通过启用我添加到 `rxjs-tslint-rules` 包里的 [`rxjs-no-unsafe-takeuntil`](https://github.com/cartant/rxjs-tslint-rules#rules) 规则来确保 `takeUntil` 是 `pipe` 中的最后一个操作符。

***

## 更新

通常的规定是将 `takeUntil` 放到最后。然而在有些情况下，你可能需要把它放到倒数第二个的位置上。

在 RxJS 的操作符中，有一些是只在源 observable 完成时才会发出值的。就比如说 `count` 和 `toArray`，只有在它们的源 observable 完成时，它们才会发出源 observable 中数据的个数，或是其组成的数组。

当一个 observable 因 `takeUntil` 而完成时，类似 `count` 和 `toArray` 的操作符只有放在 `takeUntil` 后面才会生效。

另外还有一个操作符是你需要放在 `takeUntil` 后面的，那就是 `shareReplay`。

目前的 `shareReplay` 有一个 bug/feature：它永远不会取消其源 observable 的订阅，直到源 observable 完成，或是发生错误，详见 [PR](https://github.com/ReactiveX/rxjs/pull/4059)。所以将 `takeUntil` 放在 `shareReplay` 后面是无效的。

上面提到的 TSLint 规则是[知道这些例外](https://github.com/cartant/rxjs-tslint-rules/blob/v4.14.2/source/rules/rxjsNoUnsafeTakeuntilRule.ts#L41-L62)的，所以你不用担心会造成一些莫名其妙的问题。

***

在 6.4.0 版本的 RxJS 中，`shareReplay` 做了一定修改，现在你可以通过 `config` 参数来指定其引用计数行为。如果你指定了 `shareReplay` 的引用计数，就可以把它安全地放到 `takeUntil` 前面了。

想了解更多有关 `shareReplay` 的信息，请看[这篇文章](https://medium.com/@cartant/rxjs-whats-changed-with-sharereplay-65c098843e95)。
