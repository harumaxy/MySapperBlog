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




