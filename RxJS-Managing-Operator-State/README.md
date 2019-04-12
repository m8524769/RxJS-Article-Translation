# [译] RxJS: 操作符状态管理

> 原文链接：[RxJS: Managing Operator State](https://medium.com/@cartant/rxjs-managing-operator-state-2f20681df21d)<br/>
> 原文作者：[Nicholas Jamieson](https://medium.com/@cartant)；发表于2019年2月12日<br/>
> 译者：[yk](https://github.com/m8524769)；如需转载，请注明[出处](https://github.com/m8524769/RxJS-Article-Translation)，谢谢合作！

![](assets/1_yHfcFpUTJiBKy6bvQFDPmA.jpeg)

> 摄影：Victoire Joncheray，来自 Unsplash

在 RxJS 5.5 引入了[管道操作符（pipeable operators）](https://blog.angularindepth.com/rxjs-understanding-lettable-operators-fe74dda186d3)之后，编写用户级（userland）操作符变得更为简单了。

管道操作符属于高阶函数（higher-order function）：即返回值为函数的函数。所返回的函数接受一个 observable 对象作为参数，并返回一个 observable 对象。所以，要创建一个操作符，你不必划分 `Operator` 和 `Subscriber`，只要写一个函数就行了。

这听起来很简单。

然而在某些情况下你需要格外小心，尤其是当你的操作符在存储内部状态时更应如此。

## 举个例子

让我们来看这样一个例子：一个将接收到的数据及其索引显示到终端的 `debug` 操作符。

我们的操作符需要维护一些内部状态：索引——每收到一次 `next` 通知时就会递增。有个很戆的办法是直接将状态存储在操作符的内部，就像这样：

```typescript
import { MonoTypeOperatorFunction } from "rxjs";
import { tap } from "rxjs/operators";

export function debug<T>(): MonoTypeOperatorFunction<T> {
  let index = -1;
  // 让我们假设不存在 map 操作符，于是我们只能用 tap 来维护内部存储中的索引
  // 该操作符的目的是为了表明：运行结果取决于状态存储的位置
  return tap(t => console.log(`[${++index}]: ${t}`));
}
```

该办法会存在许多问题，并导致一些意料之外的行为和难以定位的 bug。

## 存在的问题

第一个问题是：我们的操作符不具有引用透明（referentially transparent）性。当一个函数的返回值可以替代该函数而不影响程序运行，那么我们称这个函数是引用透明的。

让我们来看看当这个操作符的返回值被用于组成一些 observable 对象时会发生什么：

```typescript
import { range } from "rxjs";
import { debug } from "./debug";

const op = debug();
console.log("first use:");
range(1, 2).pipe(op).subscribe();
console.log("second use:");
range(1, 2).pipe(op).subscribe();
```

运行结果为：

```shell
first use:
[0] 1
[1] 2
second use:
[2] 1
[3] 2
```

好吧，我知道这很令人意外。在第二个 observable 中，索引并没有从 0 开始计数。

第二个问题是：只有在首次订阅该操作符返回的 observable 时，其行为才会是合理的。

现在，让我们多次订阅由 `debug` 操作符组成的 observable，看看会发生什么：

```typescript
import { range } from "rxjs";
import { debug } from "./debug";

const source = range(1, 2).pipe(debug());
console.log("first use:");
source.subscribe();
console.log("second use:");
source.subscribe();
```

运行结果为：

```shell
first use:
[0] 1
[1] 2
second use:
[2] 1
[3] 2
```

还是同样令人意外的结果：在第二次订阅中，索引依旧没有从 0 开始计数。

所以该如何解决这些问题呢？

## 解决方案

这两个问题都可以通过基于每个订阅的状态存储（storing the state on a per-subscription basis）来解决。以下是几种实现方法：

第一种方法是使用 `Observable` 的构造函数来创建操作符返回值（observable）。如果将 `index` 变量放入传给 `constructor` 的函数中，那么每次订阅的状态都会被独立存储。写法如下：

```typescript
import { MonoTypeOperatorFunction, Observable } from "rxjs";
import { tap } from "rxjs/operators";

export function debug<T>(): MonoTypeOperatorFunction<T> {
  return source => new Observable<T>(subscriber => {
    let index = -1;
    return source.pipe(
      tap(t => console.log(`[${++index}]: ${t}`))
    ).subscribe(subscriber);
  });
}
```

第二种方法，也是我比较喜欢的，就是使用 `defer` 来实现基于每个订阅的状态存储。如果将 `index` 变量放入传给 `defer` 的工厂函数中，它就可以按每个订阅独立存储状态。写法如下：

> 译者注：`defer()` 的参数为一个返回值为 observable 的工厂函数 `observableFactory`，详见[文档](https://rxjs-dev.firebaseapp.com/api/index/function/defer)。

```typescript
import { defer, MonoTypeOperatorFunction } from "rxjs";
import { tap } from "rxjs/operators";

export function debug<T>(): MonoTypeOperatorFunction<T> {
  return source => defer(() => {
    let index = -1;
    return source.pipe(
      tap(t => console.log(`[${++index}]: ${t}`))
    );
  });
}
```

还有个较为复杂的方法，就是使用 `scan` 操作符。`scan` 会维护每个订阅的状态，该状态由 `seed` 参数进行初始化，然后通过 `accumulator`（累加器）函数计算并返回结果。在本例中，`index` 可以像这样存储在 `scan` 中：

> 译者注：`accumulator` 和 `seed` 为 `scan()` 的两个参数，详见[文档](https://rxjs-dev.firebaseapp.com/api/operators/scan)。

```typescript
import { MonoTypeOperatorFunction } from "rxjs";
import { map, scan } from "rxjs/operators";

export function debug<T>(): MonoTypeOperatorFunction<T> {
  return source => source.pipe(
    scan<T, [T, number]>(([, index], t) => [t, index + 1], [undefined!, -1]),
    map(([t, index]) => (console.log(`[${index}]: ${t}`), t))
  );
}
```

如果用以上任意一种方法来代替一开始那个很戆的办法，输出将会是下面这样：

```shell
first use:
[0] 1
[1] 2
second use:
[0] 1
[1] 2
```

如你所愿：一切都在意料之中。
