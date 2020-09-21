---
title: rxjs勉強 5
date: "2020-09-20T07:39:43.897Z"
tags: [rxjs, javascript, subject, stor  e]
---

# Store Observable pattern

Store の概念を理解し、独自 Store を作る

# Store

Store の概念は誤用されやすい

可能な限り、`new Observable()`や、`of()`, `fromEvent(), fromPromise()`など
用意されている関数からデータソースを使って Observable を作成したほうがいい

ただし、まれに built-in の関数では扱いにくいデータソースもある。

# Subject

```ts
const subject = new Subject();
subject.next();
subject.error();
subject.complete();
subject.pipe();
```

`Subject`は、Observable の作成コールバックで出てくる `observer`と
同じ API を持っている。

`subject.asObservable()`の戻り値で、Observable を取得できる

```ts
const subject = new Subject();
const series$ = subject.asObservable();
series$.subscribe();

subject.next(1);
subject.next(2);
subject.next(2);
subject.complete();
```

Observable を作成・subscribe したあとで Observable にデータを流すことができ
宣言的ではなく手続き的に Stream にデータを追加できる。

カスタム Observable を作るのに役立つ場面もある

## Subject のデメリット

Rxjs の組み込みデータソース関数から作ることができる場合、そっちを使ったほうが better

- unsubscribe() ロジックを提供する手段がない
- アプリケーションの他の部分が Subject を共有するリスク
  - 想定していない動作でエラーになる可能性もある

`from`, `fromPromise`, `fromEvent`などで殆どの場合は事足りる

## MultiCasting

Subject の正規的な使い方の一つ
1 つの Observable からのある値を、複数の output Stream に分離したいとき

## Subject のデフォルト動作

Built in 関数から作った Observable は、
subscribe()の回数が増えると、Data Stream の数も増えた。

ただし、Subject から作った Observable は、
すでに放出された値は あとから Subscribe したら受け取れない
(no memory)

```ts
const subject = new Subject();
const series$ = subject.asObservable();
series$.subscribe();

subject.next(1);
subject.next(2);
subject.next(3);
subject.complete();

setTimeout(() => {
  series$.subscribe(console.log);
  subject.next(4);
}, 3000);
```

最初に subscribe したものは 1,2,3,4
3 秒後から subscribe したものは 4

# BehaviourSubject

ロジックと、データの来るタイミングを無関係にする Subject

```ts
const subject = new BehaviorSubject(0);
const series$ = subject.asObservable();
series$.subscribe((val) => console.log("early :" + val));

subject.next(1);
subject.next(2);
subject.next(3);

setTimeout(() => {
  series$.subscribe((val) => console.log("late :" + val));
  subject.next(4);
}, 3000);
```

- 初期値が必要
- あとに subscribe した方は、subscribe 前の最終値を初期値として受け取る
- complete するともう流れない

最も使われる Subject で、Store を実装するために使われる

# AsyncSubject & ReplaySubject

初期値がいらない
初期値が必要なのは BehaviourSubject だけ？

## AsyncSubject

`subject.complete()`が発生したあとに、最後の値だけ `subscribe()`にわたす
HTTP リクエストなどに最適

## ReplaySubject

`subject.next()`した値を、あとから subscribe()したときにも渡す

# Store Service Design

Subject を一般的なデザインパターンを実装して実用化する。

現状だと、HTTP リクエストの結果をクライアント側メモリに保存していないので、
進む > 戻る を繰り返すと
コース一覧ページに戻るたびに HTTP リクエストが発生する。

centered Store を実装して、リクエストを再送信することを抑制する.

## centored Store

React, Angular, Svelte, Vue など、
ページ遷移のときにコンポーネントを差し替える実装をしているとき、

- 複数ページで共有する
- アプリ通して全体で永続化する

といったデータは、コンポーネントに State として持つのではなく、
centered Store に格納して使い回すのが良い
(Data を一元化した Store Service にする)

名前は `store.****.ts` (Angular 特有の命名？)
他のフレームワークだったら自由につけてもいいかも

`Subject`を使って Observable を作り、
HTTP 通信を行うタイミングを制御する
