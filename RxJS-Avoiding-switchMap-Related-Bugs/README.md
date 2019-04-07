# [译] RxJS: 避免因滥用 switchMap 而导致错误

> 原文链接：[RxJS: Avoiding switchMap-Related Bugs](https://blog.angularindepth.com/switchmap-bugs-b6de69155524)<br/>
> 原文作者：[Nicholas Jamieson](https://blog.angularindepth.com/@cartant)<br/>
> 译者：[yk](https://github.com/m8524769)；
> 翻译内容有些部分借鉴了 [vannxy](https://link.zhihu.com/?target=https%3A//github.com/vaanxy) 的 [RxJS: 避免 switchMap 的相关 Bug](https://zhuanlan.zhihu.com/p/55199532)，深表感谢

不久前，[Victor Savkin](https://medium.com/@vsavkin) 发布了一篇关于在 Angular 应用中使用 NgRx effects 时，因滥用 `switchMap` 而导致的一个不易察觉的 bug 的推文：

> [**Victor Savkin**](https://twitter.com/victorsavkin)<br/>
> _@victorsavkin_
>
> 我看过的每个 Angular 应用都因为 switchMap 的使用不当而产生很多错误。这是跟 RxJS 有关的 issues 的最大来源。 [#angular](https://twitter.com/hashtag/angular?src=hash)

## 那么这个 bug 是什么呢？

让我们以购物车为例，看看下面的 effect 和 epic 是怎么滥用 `switchMap` 的，然后我们再考虑用一些替代的操作符。

这是一个滥用 `switchMap` 的 NgRx effect：

```typescript
@Effect()
public removeFromCart = this.actions.pipe(
  ofType(CartActionTypes.RemoveFromCart),
  switchMap(action => this.backend
    .removeFromCart(action.payload)
    .pipe(
      map(response => new RemoveFromCartFulfilled(response)),
      catchError(error => of(new RemoveFromCartRejected(error)))
    )
  )
);
```

这是一个与之等价的 `redux-observable` epic：

```typescript
const removeFromCart = actions$ => actions$.pipe(
  ofType(actions.REMOVE_FROM_CART),
  switchMap(action => backend
    .removeFromCart(action.payload)
    .pipe(
      map(response => actions.removeFromCartFulfilled(response)),
      catchError(error => of(actions.removeFromCartRejected(error)))
    )
  )
);
```

我们的购物车列出了用户打算购买的商品，每个商品都有一个“移出购物车”按钮。点击该按钮就会将 `RemoveFromCart` 动作调至 effect/epic ，后者与后端通信并移除该商品。

大多数情况下，这都将如期运行。然而，使用 `switchMap` 会在这里引入竞争条件（race condition）。

如果用户一连点击了多个商品的“移出购物车”按钮，则结果将取决于按钮点击的频率。

如果用户在 effect/epic 与后端通信时再次点击了按钮，之前的移除操作就会被挂起，而在这使用 `switchMap` 则会中止之前被挂起的操作。

因此，根据按钮点击的频率，程序可能会：

- 删除所有被点击的商品；
- 仅删除部分被点击的商品；
- 在后端删除了部分被点击的商品，而前端的购物车却没反应。

很明显，这是一个 bug。

不幸的是，当需要使用 flattening operator（打平操作符）时，`switchMap` 通常会被建议是首选，但这并不是在所有场景下都是安全的。

RxJS 共有四个 flattening operator 可供选择：

- `mergeMap`（也称为 `flatMap` ）；
- `concatMap`；
- `switchMap`；
- `exhaustMap`。

让我们来看看这些操作符，了解它们之间的差异，并决定哪个操作符最适合购物车场景。

## mergeMap/flatMap

如果将 `switchMap` 替换为 `mergeMap`，effect/epic 将同时处理每个已调来的操作。也就是说，被挂起的移除操作将不会被中止；后端请求会被同时发起，当移除完成后再处理响应。

需要重点注意的是，由于操作是并发处理的，所以响应的顺序可能与请求的顺序不同。举个例子，如果用户依次点击了两个商品的“移出购物车”按钮，后点击的商品可能会先被移除。

在购物车场景里，商品的移除顺序并不重要，所以使用 `mergeMap` 来代替 `switchMap` 可解决这个问题。

## concatMap

虽然从购物车中移除商品的顺序可能无关紧要，但也有一些操作对执行顺序是有严格要求的。

再举个例子，如果我们的购物车有一个用于增加商品数量的按钮，则以正确的顺序来处理调度的操作是非常重要的。否则，前后端的商品数量可能最终会不同步。

对于顺序十分重要的操作，我们应该使用 `concatMap`，其实 `concatMap` 就相当于使用 `mergeMap` 时将其“允许并发量”参数 `concurrent` 设置为 `1` 。也就是说，使用 `concatMap` 的 effect/epic 每次只会处理一个后端请求，每个操作都会按照它们被调用的顺序排队。

`concatMap` 是安全且保守的选择。如果你不确定在 effect/epic 中使用何种 flattening operator 时，就用 `concatMap` 吧。

## switchMap

每当相同类型的操作被调用时，使用 `switchMap` 会中止之前已被挂起的后端请求。这使 `switchMap` 对于增加、修改以及删除操作来说不那么安全。甚至在处理读操作时也会引入 bug 。

`switchMap` 是否适合特定的读操作取决于当另一个相同类型的操作被调用时，后端对先前操作做出的响应是否还有用。让我们来看看一个使用 `switchMap` 的操作是如何引入 bug 的。

如果我们的购物车中的每个商品都有一个“详情”按钮，用于在行内显示一些商品的详细信息，effect/epic 则使用 `switchMap` 来处理该按钮的点击动作，这里又引入了一个竞争条件。如果用户一连点击了多个商品的“详情”按钮，那么这些被点击的商品是否会显示详细信息则同样取决于用户点击按钮的频率。

和 `RemoveFromCart` 操作一样，使用 `mergeMap` 就可以解决这个问题了。

`switchMap` 只应当用于 effects/epics 中的读操作处理，并且当另一个相同类型的操作被调用时，后端对先前操作做出的响应可被丢弃。

让我们来看看 `switchMap` 的一个适用场景。

如果我们的购物车要显示商品的总价加上运费，对购物车内容做出的每个更改都会触发一次 `GetCartTotal` 操作。这时在 effect/epic 中使用 `switchMap` 来处理 `GetCartTotal` 是完全合适的。

如果当 effect/epic 正在处理一个 `GetCartTotal` 操作时，购物车的内容发生了变动，那么对当前处理中的请求做出的响应将会是过时的，也就是更改之前购物车内的商品总数，因此中止正在处理中的请求是没有任何问题的。事实上，相较于挂起请求，等其完成之后再将其忽略，或更有甚者把过期的响应数据也渲染出来，直接中止请求会是更好的选择。

## exhaustMap

`exhaustMap` 或许是最不为人知的一个 flattening operator 了，但它很好解释：你可以认为它是 `switchMap` 的对立物。

当使用 `switchMap` 时，之前挂起的后端请求会被中止，也就是说更倾向于处理最新调来的操作。而当使用 `exhaustMap` 时，如果当前有正在处理的后端请求，那么新调来的操作都会被忽略。

让我们来看看 `exhaustMap` 的一个适用场景。

开发人员应该对有一类用户再熟悉不过了：按钮狂击者。当他们点击一个按钮却发现什么都没发生时，就会继续点点点一直点。

假如我们的购物车有一个刷新按钮，effect/epic 使用 `switchMap` 来处理刷新，那么每次点击按钮都会中止先前被挂起的刷新操作。所以狂点按钮没有任何意义，而且可能使用户等待更长的时间，直到刷新被执行。

如果在 effect/epic 中使用 `exhaustMap` 来替代 `switchMap` 处理刷新的话，多余的点击将会被忽略。

## 总结

总而言之，当你需要在一个 effect/epic 中使用 flattening operator 时，你应该：

- 当操作不能被中止/忽略，且必须保证执行顺序的情况下，使用 `concatMap`；（这也是一个保守的选择，因为其总会按照预期运行）
- 当操作不能被中止/忽略，但执行顺序并不重要的情况下，使用 `mergeMap`；
- 当处理读操作时，且当另一个相同类型的操作被调用时，先前的请求应当被中止的情况下，使用 `switchMap`；
- 当相同类型的操作应当被忽略的情况下，使用 `exhaustMap`。

## 使用 TSLint 来避免滥用 switchMap

我曾在 `rxjs-tslint-rules` 包里添加了一条 [`rxjs-no-unsafe-switchmap`](https://github.com/cartant/rxjs-tslint-rules) 规则。

该规则可以识别 NgRx effects 和 `redux-observable` epics，并确定它们的操作类型，然后根据操作类型搜索具体的动词（例如：`add`，`update`，`remove` 等等）。它有一些合理的默认值，如果你觉得这些默认值过于常规，也可以对它进行[配置](https://github.com/cartant/rxjs-tslint-rules#rxjs-no-unsafe-switchmap)。

在启用了该条规则后，我在我去年写的一些应用上运行了 TSLint ，发现了不少 effects 都在以不安全的方式使用 `switchMap`。所以，谢谢你的推文，Victor。
