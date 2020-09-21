---
title: rxjs勉強 1
date: "2020-09-15T14:17:38.747Z"
tags: [rxjs, javascript, observable, timer, interval]
---

# Stream

JS の世界では様々なイベントが起こる

- click: クリックしたら発生
- interval: 定期的に発生
- timeout: 一定時間後に 1 回だけ発生

これらはそれぞれ `Stream` の例足り得る

```js
document.addEventListener("click", (event) => {
  console.log(event);
});

let counter = 0;

setInterval(() => {
  console.log(counter);
  counter++;
}, 1000);

setTimeout(() => {
  console.log("finished...");
}, 3000);
```

今の状態では、3 つの分けられた Free Streams

## Callback Hell

さっきの 3 つを入れ子にした状態
クリックのたびにインターバルが増えるので、
イベントの回数がとんでもなく増える

```js
document.addEventListener("click", (event) => {
  console.log(event);

  setTimeout(() => {
    console.log("finished...");

    let counter = 0;

    setInterval(() => {
      console.log(counter);
      counter++;
    }, 1000);
  }, 3000);
});
```

click, timeout, interval, その他 asyncronous なイベントは組み合わさって
複雑な Stream になることが多い。
(Nested な Callback)

上の例だと、クリックした回数だけ interval streams が並行で発生する。

ブラウザレベルで callback を登録するより、Rxjs で
Stream を combine して simple にすることで、maintanable に扱える。

# What is Observable ?

かんたんな interval observable を作る。
変数名の最後に `$` をつけるのが observable の通例

```js
import { interval } from "rxjs";

const interval$ = interval(1000); // Observable<number>
```

`<number>`型の Observable

Observable オブジェクトを作っただけでは、実際には `Stream` の値は放出されない。
これは、 `Stream` of values の定義。
Stream がインスタンス化されたときにどのように振る舞うかの設計図

## timer(delay, interval)

interval の、最初に待ち時間入れるバージョン
3 秒待ったあと、1 秒ごとに値を放出する `Stream`

```js
const interval$ = timer(3000, 1000);

interval$.subscribe((val) => console.log("stream 1 " + val));
```

# observable.subscribe()

`Observable`オブジェクトは、`Stream`の設計図。
`subscribe()`することで、実際に`Stream`が作成される。

```js
interval$.subscribe((val) => console.log("stream 1 " + val));
```

`Stream`を増やすには、その分だけ`subscribe`を実行する。
下の例は、2 つのストリームを並行して処理

```js
interval$.subscribe((val) => console.log("stream 1 " + val));
interval$.subscribe((val) => console.log("stream 2 " + val));
```

## fromEvent()

ブラウザの Native API から発生する event で Observable を作る。
`fromEvent(target, "event_name")`

```js
import { fromEvent } from "rxjs";

const click$ = fromEvent(document, "click");
click$.subscribe((event) => console.log(event));
```

ここでは、`document`全体のクリックイベントを拾っているが
場合によっては `button`要素や`canvas`要素などのイベントになったりする。

### `Observable<T>` の違い

`interval/timer()` で作ると `Observable<number>`
`fromEvent()`で作ると `Observable<Event>`

Observable はジェネリック型であり、どんな値でも`Stream`にできる

## ここまでまとめ

- 時間変化、イベントによって発生する値の流れを `Stream` という
- `Observable<T>` は `Stream` の設計図
- `observable$.subscrive(func)` で実際にストリームが作成される。

# Subscribe Method

`observable$.subscrive(cb1, cb2, cb3)`

- `val => ...`: `Stream`の値を処理する
- `err => ...`: エラーを処理する
- `() => ...;`: 何も値を返さないときの最後の処理

Subscribe のコールバックでエラーが発生すると、それよりあとには`Stream`が流れない。
なので 3 番目のコールバックが最後に実行される

```js
interval$.subscribe(
  (val) => console.log(val),
  (err) => console.log(err),
  () => console.log("completed")
);
```

## Cancelation (unsubscribe)

`observable$.subscribe(func)` メソッドを実行すると、`Subscription`オブジェクトを返す。
それを任意のタイミングで`subscription.unsubscribe()`することで、`Stream`の処理を中止することができる。

途中で値の処理をやめたくなったときに有効 ß

```js
const interval$ = timer(3000, 1000);

const sub = interval$.subscribe((val) => console.log("stream 1 " + val));
setTimeout(() => sub.unsubscribe(), 5000);
```
