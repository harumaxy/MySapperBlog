---
title: rxjs勉強 1
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

# concatMap & saveObservable

