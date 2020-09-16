---
title: rxjs勉強 2
date: "2020-09-15T15:24:10.094Z"
tags: [rxjs, javascript]
---

# How Observables Work Under the Hood

Observable が実際はどのように動いているのかを見る。

# Build Own HTTP Observable

これまで、`interval` `timer` `fromEvent` など
Rxjsが提供する関数から`Observable<T>`を作成してきた。

上に当てはまらない独自のObservableを作る(例えばHTTP通信、その他非同期処理)

そのためには、`Obserbable`をコンストラクターから作る
(動画だと`Obserbable.create()`を使ってたけど非推奨らしい)


```js
const http$ = new Observable((observer) => {
  observer.next();
  observer.error();
  observer.complete();
});
```

`obserber` (本当は subscriber という名前)のオブジェクトを引数に取るコールバックを使って作る

`observer.`
- `next(arg)`
- `error(err)`
- `complete()`

`observer`オブジェクトの持つメソッドに引数を適用すると、
`Observable.subscribe(cb)` に登録したコールバックに引数が渡される。



## 実際にやってみる

```js
const http$ = new Observable((observer) => {
  fetch("/api/courses")
    .then((response) => response.json())
    .then((body) => {
      observer.next(body);
      observer.complete();
    })
    .catch((err) => observer.error(err));
});
```

ここで作られた `http$` Observable は
まだ設計図なので、`Stream`は作成されてない 
= この式を評価しても`HTTP Request`は実行されない。

`subscribe`することで、実際にリクエストされる
```js
http$.subscribe(
  (courses) => console.log(courses),
  noop,
  () => console.log("completed")
);
```
`noop = ()=>{}`, 何もしない関数
noop は Rxjs が用意している関数


## HTTP fetch promise を Observable にするメリット

Observable の形になっていると、`Rxjs Operator`を使って他の`Stream`に変換したり、組み合わせたりできる。
例えば `fetch`の結果の中に`Array`があったりしたら、それを`Stream`に変換して
次の処理に流せる。

