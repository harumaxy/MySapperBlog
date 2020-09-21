---
title: rxjs勉強 4
date: "2020-09-20T07:39:43.897Z"
tags: [rxjs, javascript, error_handling, custom_operator, forkJoin]
---

# switchMap

https://rxjs.dev/api/operators/switchMap

flatMap + merge +
前のやつが取得できる前に新しい入力値を受け取ったら前のやつをキャンセルする。

リンクの例だと、

- 10 は全部出力できたので流す
- 30 は途中で 50 の入力が入ってきたのでキャンセル

HTTP 通信に置き換えると、前のリクエストが完了する前に
新しいリクエストを始めたら前のやつはキャンセルするイメージ

前の講義で作った `createHTTPObservable()` で
`unsubscribe()`処理で fetch をキャンセルしている

# Error Handling with Reactive Programming

エラーを処理するには、Operator を挟む
Promise と同じように、 catch でエラーを処理して
代わりのデータを流すことで処理を継続できる

## catchError Operator

Operator の基本は、 `Observable` を引数にして `Observable`を返す関数。
なので、catchErro でも Observable を返す

```ts
observable$.pipe(
  catchError(err => of([1,2,3...])),
)
```

または、値を返さないで error を throw することもできる

```ts
observable$.pipe(
  catchError(err => of([1,2,3...])),
)
```

その場合、エラーハンドリングは別の関数が行う

## finalize Operator

エラーがハンドリングされなかったとき
または Observable のデータがすべて放出し終わったあと
に実行される

```ts
observable$.pipe(
  catchError(err => of([1,2,3...])),
  finalize(() => console.log("some error occered"))
)
```

要は、onCompleted()の operator 版

## Retry Error Handling

失敗したらもう一回やる

```ts
const isError = Math.random() >= 0.5;
if (isError) {
  res.status(500);
} else {
  res.status(200);
}
```

サーバーサードがこういう実装になってるとする。
1/2 の確率でエラーになる

## retryWhen Operator

`Observable<error> => Observable<any>`
エラーの Observable を受け取り、
inteval/timer 系の Observable を返すことで、
指定時間後にリトライする。

`~~~When`系は、時間経過系 Observable を返す

```ts
observable.pipe(retryWhe((errors) => errors.pipe(timer(2000))));
```

## startWith Operator

初期化処理をかんたんにする。

`Ovesarvable<any> => Observable<any>`

これを pipe で挟んだところは、
どの値よりも早く最初の一回が指定した値を返すようになる

```ts
observable$.pipe(
  map(event => event.target.value),
  startWith(''),
  ...
)
```

どんな値が入ってこようとも、最初の一回だけは空文字列でコースを検索して表示する。

# throttle

https://www.learnrxjs.io/learn-rxjs/operators/filtering/throttle

https://rxjs.dev/api/operators/throttle

throttle = 絞り、スロットル、燃料の絞り弁を調節するレバー

`debounceTime`の先頭版
durationSelector 引数で指定した期間のうち、最初の入力は受け取るが
その次以降の入力は期間が終わるまで無視する。

ストリーム内の値を減らす意味合いを持つ。

```ts
fromEvent<any>(this.input.nativeElement, "keyup")
  .pipe(
    map((event) => event.target.value),
    throttle(() => interval(500))
  )
  .subscribe();
```

## throttleTime

`throttle + interval`
throttle には ()=>interval をコールバックで渡すが、
それが最初からセットになっている operator

```ts
throttleTime(500) = throttle(() => interval(500));
```

### debounce vs throttle

debounce は、入力を終えるまで値を送信しない
代わりに、値は必ず最新値
Streme の値の数はかなり少なくなる

throttle は、入力感覚を絞るだけで入力状況自体は時間ごとに送信
ただし、値が最新値とは限らない

# Custom Operator

基本的には、
`Observable<any> => Observable<any>`
の関数を作ればいい
そうすれば pipe で使える

## 例. debug

```ts
enum level = {INFO, TRACE, ERROR, ...};

export const debug(level, message) =>
  (src: Observable<any>) =>
    source.pipe(
      // いろいろな operator を詰め込んで自分の独自operator にする
    )
```

メリット = いろんなところで重複する処理を共通化する。
logging などはその最たるものの一つ

# Stream をまとめる

## forkJoin 関数

`(Observable<T>, Observable<S>) => (Observable<[T, S]>)`
複数の Observable をタプルにまとめる
すべての Observable が完了するまで待ってから、それぞれの最後の値を出力する。

いわゆる、`Promise.all()` と同じ

最後の値だけでなく、途中の値を結合して取得したいときは `combineLatest()`, `zip()`

使用例 `courses` `lessons` をそれぞれリクエストして、どっちも取得してから表示
