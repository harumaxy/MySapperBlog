# sapper-blog-template


tailwind + postcss の導入
https://dev.to/alancwoo/sapper-svelte-postcss-tailwind-473i

テンプレート使ったほうが早い


# Typescript化

ここを参考に
https://neos21.hatenablog.com/entry/2020/09/08/080000


# Trouble shoot

## implicitly using "default"

```
Entry module "rollup.config.js" is implicitly using "default" export mode, 
```

ビルド中にこれが出たら、`rollup.config.js` の `export default` を `module.exports` に変えてやると消える


## img aledy known

`img` タグの `alt` attribute に img, image などの単語が入っていると、
すでに画像であることは知っているので消したほうが良いという警告が出る。


## front matter を追加する。

1. 記事のフロントマターを増やす
2. `src/utils/markdown.js` の中身を弄って、`gray-matter`モジュールに解釈される戻り値を増やす
  - このモジュールは`rollup.config.js`で読み込まれ、`*.md`をES6モジュールとして読み込んだときにオブジェクトとしてパースする。
3. `src/routes/blog/_posts.js`ですべてのコンテンツを`JSON.stringify()`しているところもいじる
  - 並び順も変えられそう。
  - `all`以外にも、タグ順とか古い順とかできそう



## blog の記事をフォルダごとにまとめる

画像、コード、その他をひとまとめにすると便利

markdownを ESモジュールとして読み込むときに、以下のように書けばファイルの階層関係なくすべて配列として取れる。

```js
import all from "./posts/**/*.md";
```

## markdown

`mdsvex`を使う。
これを使うと markdown が svelte コンポーネントに変換される
(プリプロセッサ)

`.svx` 拡張子を使う(`.svelte`にも書けるけど)

例にもれず `rollup.config.js` の設定がだるい

[JavaScript in Markdown ✅](https://sapper-goals.netlify.app/goals/javascript-in-markdown/)



## Netlify で CSS の読み込みに失敗する

`static/assets` フォルダを作って、そっちに移動して
`template.html`の参照パスも更新したら直った。


https://stackoverflow.com/questions/36545803/failed-to-load-css-file

なんでかはよくわからん。