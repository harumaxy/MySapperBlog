---
title: rxjs勉強 3
date: "2020-09-16T13:37:47.619Z"
tags: [rxjs, javascript]
---

# Rxjs Operators

http://reactivex.io/documentation/operators.html

`ReactiveX`の文脈での `Operator` は、
`Observable` を操作して `Observable`を返す。

更に、Operator は Chain することができる。
関数型言語のパイプ演算子みたいなもの

# Map
Operator の一つ
Observable に放出された値を加工して返す。

List.map とかと同じ

## pipe

`observable$.pipe(op)`
`.pipe`メソッドは、Operator オブジェクトを引数に取る
Observable から流れて来る値を引数にした operator にわたす
変換された値がまた Steam に流される

## map(cb)

関数を引数にとって、`Map` Operator を作る関数
Rxjsで提供されている　

```ts
const http$ = createHttpObservable("/api/courses");
const courses$ = http$.pipe(map((res) => Object.values(res["payload"])));
```

実際に変換された値を使うには、`.subscribe()`
```ts
const http$ = createHttpObservable("/api/courses");
    const courses$ = http$.pipe(
      map((res) => Object.values<Course>(res["payload"]))
    );

    courses$.subscribe(
      (courses) => {
        this.beginnersCourses = courses.filter(
          (course) => course.category == "BEGINER"
        );
        this.advancedCourses = courses.filter(
          (course) => course.category == "ADVANCED"
        );
      },
      noop,
      () => console.log("completed")
    );
``` 

## ここまでまとめ

- Observable = Stream の設計図
- Operator = Observable を操作して Observable を返す
  - observable.pipe(op)
    - 関数型言語のパイプみたいな感じ
- Map
  - Observable に関数を適用して値を変換する Operator
- subscribe(cb, cb, cb)
  - Observable.pipe(map).pipe(op)....subscribe()
  - operator で変換されてきた Stream 値の最終処理場
  - 副作用的操作をする場所？

### Observable の作成を関数にする

いわゆる Builder Pattern
オブジェクトを作る関数を作って、モジュールとして分離する

```ts
//util.ts
export const createHttpObservable = (url: string) =>
  new Observable((observer) => {
    fetch(url)
      .then((response) => response.json())
      .then((body) => {
        observer.next(body);
        observer.complete();
      })
      .catch((err) => observer.error(err));
  });

```


# Reactive Designe

さっきのコードで、`.sbuscribe()`の中で
渡されてきた array を filter して `begginerCourses` と `advancedCourses`
に分けた。

ただ、そもそも Stream を 1つしか作っては行けない決まりはないので、
どちらも `Observable<Course>` に置き換える。


```ts
export class AboutComponent implements OnInit {
  beginnersCourses$: Observable<Course[]>;
  advancedCourses$: Observable<Course[]>;

  ngOnInit() {
    const http$ = createHttpObservable("/api/courses");
    const courses$ = http$.pipe(
      map((res) => Object.values<Course>(res["payload"]))
    );

    this.beginnersCourses$ = courses$.pipe(
      map((courses) => courses.filter((course) => course.category == "BEGINER"))
    );
    this.advancedCourses$ = courses$.pipe(
      map((courses) =>
        courses.filter((course) => course.category == "ADVANCED")
      )
    );
  ...
  }
}
```

`course$.subscribe()` の中でセットするパターンと比べると、
ネストが2つくらい少なくなり、保守しやすい感じ。
同じように機能するが、監視可能な Data Stream の概念でプログラム全体を構築できる。


## shareReplay Operator

Rxjs で HTTPリクエストを扱うときに使われることの多い operator


実は、上のコードでは `/api/courses`へのアクセスが2回起こる
`http$ -> courses$`
- `-> beginnerCourses`
- `-> advancedCourses`

data stream を 2つ作る関係上、源流にあるhttpリクエストも2回行われてしまう

Observable は Stream の設計図であり、`subscribe()`を2回行うと Stream も2つ作られる。
同じデータを使う場合、メモリ効率上良くない。


```ts
const courses$ = http$.pipe(
  tap((val) => console.log("http requested", val)),
  map((res) => Object.values(res["payload"]) as Course[]),
  shareReplay()
);

courses$.subscribe()
this.beginnersCourses$.subscribe();
this.advancedCourses$.subscribe();
```

`subscribe()`すると、
`shareReplay()` Operator の直前のストリームが
要求するたびにリプレイされる。
上の例だと `tap()`は一回しか実行されない。
(位置を`shareReplay()`の下にすると`subscribe()`のたびに実行される)


HTTPリクエストを Stream に組み込む場合、何度もリクエストすると
負荷がかかるので、必ず`shareReplay()`を挟むべし

## Tap Operator
Stream の処理の途中で何らかの副作用的処理を挟む Operator
引数に取った callback で何か値を返しても、Stream の値は変わらない


# Higher Order Rxjs Mapping Operators

- switchMap
- mergeMap
- concatMap
- exhaustMap

# Observable Concatnation

http://reactivex.io/documentation/operators/concat.html

Observable の連結

2つの Stream を連結して、1つの Stream にする
正確には 2つの Observable を 1つにする。

## concat()
複数引数を取る関数
Observable を連結する。
順番は最初の引数から。

```ts
const source1$ = of(1, 2, 3);
const source2$ = of(4, 5, 6);
const source3$ = of(7, 8, 9);
const result$ = concat(source1$, source2$, source3$);

result$.subscribe(console.log);
```

## 重要な終了の概念

`onCompleted()`に到達しない Observable を連結した場合、
後ろに連結した Observable の値は永遠に来ない

```ts
const source1$ = interval(1000);
const source2$ = of(4, 5, 6);
const source3$ = of(7, 8, 9);
const result$ = concat(source1$, source2$, source3$);

result$.subscribe(console.log);  // -> 0, 1, 2, 3, 4, 5, ...
```

## observable concat のメリット

フォーム入力の自動保存機能

## filter() Operator

boolを返す引数を callback に渡す
true なら Stream を流す
false ならそこで取り除かれる

```ts
ngOnInit() {
  this.form.valueChanges.pipe(filter((val) => false)).subscribe(console.log);
}
```

上のコードだと、フォームの値に変化があってもコンソールに出力されない
コレを利用して、バリデーションを仕込み
正しいときは保存、間違っているときは保存しない
というふうにできる

## 例. validation が成功したら PUT メソッド

```ts
ngOnInit() {
  this.form.valueChanges
    .pipe(filter(() => this.form.valid))
    .subscribe((changes) => {
      fetch(`/api/courses/${this.course.id}`, {
        method: "PUT",
        body: JSON.stringify(changes),
        headers: {
          "content-type": "application/json",
        },
      });
    });
}
```

コレでも良いけど、ちょっとネストが深い
HTTPリクエスト部分もObservableにする
(Reactive Style)

## fromPromise()
Promise値を引数にして Observable にする関数
`new Observable(subscribe => {...})`で作るよりかんたん
多分、err とかもちゃんと onError に適用しといてくれる。

フォームの入力を保存リクエストする部分をObsrvable化
更にメソッドとして分離

```ts
this.form.valueChanges
  .pipe(filter(() => this.form.valid))
  .subscribe((changes) => {
    saveCourse$.subscribe();
  });

saveCourse(changes) = fromPromise(
  fetch(`/api/courses/${this.course.id}`, {
    method: "PUT",
    body: JSON.stringify(changes),
    headers: {
      "content-type": "application/json",
    },
  })
);

```

## 問題点
1文字更新されるたびに`PUT`メソッドでリクエストしてしまう。
ぶっちゃけとんでもない量のHTTPリクエスト。

変更が始まって、キーボードの入力が終わってから数秒後に
まとめてリクエストする形にすれば、APIを叩く回数を減らせる。

コレを、`Obserbable Concatnation`で実現する。

# concatMap()

実装よっては FlatMap とも呼ばれる
http://reactivex.io/documentation/operators/flatmap.html
https://www.learnrxjs.io/learn-rxjs/operators/transformation/concatmap


Observable によって発行された値を使ってPromiseを返す関数を map する。
Promiseは resolve された順番で Stream に流される。


# Merge Obaservable combination strategy

Observable の統合の組み合わせ戦略

Merge strategy

# Merge Operator

http://reactivex.io/documentation/operators/merge.html

複数の Observable を一つに結合する

Concat との違いは、
- Concat -> 順番につなぐ、直列処理
- Merge -> 2つのStreamを同時に流す、並列・並行処理

```ts
const interval1$ = interval(1000);
const interval2$ = interval1$.pipe(map((val) => 10 * val));

const result$ = merge(interval1$, interval2$);
result$.subscribe((val) => console.log(val));
```


## HTTP 通信における concatMap vs mergeMap

- concatMap: Promise 値を、元の Stream の順番と同じ順番になるように解決する
- mergeMap: 並行処理で実行して、解決した時間順で stream に流れる
  - 大体はHTTP通信を開始した時間順になるが

イメージ
```sh
# concatMap
---
   ---
      ---

# meregMap
---
 ---
  ---
```

順番や確実性が必要なときは concatMap
平行性やパフォーマンスが重要なときは mergeMap

```ts
ngOnInit() {
  this.form.valueChanges
    .pipe(
      filter(() => this.form.valid),
      mergeMap((changes) => this.saveCourse(changes))
    )
    .subscribe();
}

ngOnInit() {
  this.form.valueChanges
    .pipe(
      filter(() => this.form.valid),
      mergeMap((changes) => this.saveCourse(changes))
    )
    .subscribe();
}
```

# ExhaustMap Operator

exhaust = 使い切る、出し切る、絞り切る、排気する

`concatMap`や`mergeMap`と同じく、 stream に関数を map して
Promise 値を返すコールバックから作る　Operator

入力されたPromsie を解決して Stream に流すまで、
他の入力値は無視される

```sh
# exaustMap
---
 x
  x
   ---
    x
     x
      ---
       x
        x
```
3秒かかるPromise
1秒後と2秒後の入力値は無視
3秒後に解決されるので、新しい入力値を受け取る。

例えば、`SAVE`ボタンを連打すると、
concatMap や mergeMap だと連続でPromiseが作成されるが、
`exahstMap` + `unsubscribe`とかすれば 1回だけの入力にできたりする。


# Unsubscription

HTTPをキャンセルする。

検索ワード変更機能の実装などに便利
検索を実行中に、ユーザーが別の検索ワードを入力して
再度検索を実行したら、実行中のHTTPリクエストをキャンセルして別のリクエスト

## fetch をキャンセルする。

- `AbortController`: lib.dom.d.ts に入っている。fetch や DOM操作をキャンセルするコントローラー
- `abortcontroller.signal` : シグナル

`fetch` はキャンセルに対応する。
第２引数の config object に `{signal: abortcontroller.signal}`を渡す


## Observable コンストラクタのコールバックの戻り値

```ts
export const createHttpObservable = (url: string) =>
  new Observable((observer) => {
    const controller = new AbortController();
    const signal = controller.signal;
    fetch(url, { signal })
      .then((response) => response.json())
      .then((body) => {
        observer.next(body);
        observer.complete();
      })
      .catch((err) => observer.error(err));

    return () => controller.abort();
  });
```

new Observable で自前の Observable を作る場合は、
関数を返す事ができる。

コレが `subscription.unsbscribe()`メソッドの中身になる


```ts
const http$ = createHttpObservable("/api/courses");
const sub = http$.subscribe(console.log);
setTimeout(() => sub.unsubscribe(), 0);
```

subscribe してから 0秒で unsubscribe
何も表示されない

# type ahead search の実装
type ahead search = incremental search

いろんな呼び方がある
- autocomplete
- search as you type
- filter/find as you type (FAYT)
- incremental search
- typeahead search
- inline search
- instant search
- word wheeling


## keyup event と HTTP リクエストを直接結びつける問題点

すなわち、検索回数が圧倒的に多くなってしまうこと
H
He
Hel
Hell
Hello
Hello,
Hello,W
...

一文字入力するごとに検索するとAPIに負荷が高い
ユーザーの入力が安定してから自動検索する、`debounceTime` を設けることにする

# debounseTime Operator

https://rxjs.dev/api/operators/debounceTime

いわゆる rate limit operator
入力が止まったあとに、設定したdelay time経過したあと
最後の値が出力される。

delay time 中に新しい入力があった場合、delay をリセットして
古い値は消える。

単位がミリ秒なので注意

# distinctUntil Operator

この Operator を挟むと、
Stream で連続した値が 2回連続できた場合、重複したものを取り除いて1つにする。
incremental 検索でも、同じワードを2連続で検索する必要はないので
挟んでおく