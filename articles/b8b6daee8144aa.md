---
title: "Next.jsのRewritesルールで複雑なルーティングに立ち向かう"
emoji: "🗺️"
type: "tech"
topics: ["nextjs", "typescript"]
published: true
---

https://github.com/aiji42/rewrites-sample

## モチベーション
Next.jsのデフォルトのダイナミックルーティングでは対応しづらい複雑なルーティングパターンに対応する。

別のフレームワークで動かしていた中規模以上のサービスを Next.js にリプレイスするというシーンではよく起こる。

### 例えば次のようなケース
(都道府県 > 市区町村 > 物件ラベル のような構造で物件の一覧を表示するサイト)
- 各ディレクトリには識別子となるプレフィックスがついている
- ディレクトリデータによっては数字のみを許容するようなものもある
- 途中のディレクトリが省略されるようなケースが存在する
 (下の例の場合東京都の全市区町村内のワンルーム物件)
![](https://storage.googleapis.com/zenn-user-upload/stfevhobjp2widsp44iq5vbaq7n8)

デフォルトのダイナミックルーティングのみで対応する場合、 `pages/[...paths]/list.tsx` このようなディレクトリ構成で対応することができる。
https://nextjs.org/docs/routing/dynamic-routes

しかし、各`params`からプレフィックス部分を取り除いたり、バリデーション(例えば、cityIdが数字であるかどうかなど)を記述してやる必要があり、処理が肥大化することは避けられない。
また、最終的に一つのページファイルにまとまっているがゆえに、各パスごとにコンテンツを出し分けたいケースでは、分岐を行うロジックをも書く必要がある。

このようなケースでは、rewritesルールを活用することで、上にあげたような処理の記述を省くことができる。
https://nextjs.org/docs/api-reference/next.config.js/rewrites

## How To
冒頭に上げたURLパターンを rewrites ルールで実現する例
```js
// next.config.js
const rewrites = async () => {
  return [
    {
      // 都道府県の全物件リスト ①
      source: '/prefecture-:prefecture(\\w+)/list',
      destination: '/prefecture/:prefecture/list'
    },
    {
      // 都道府県x市区町村内の物件リスト ②
      source: '/prefecture-:prefecture(\\w+)/city-:city(\\d+)/list',
      destination: '/prefecture/:prefecture/city/:city/list'
    },
    {
      // 都道府県x絞り込みラベルの物件リスト ③
      source: '/prefecture-:prefecture(\\w+)/label-:label(\\w+)/list',
      destination: '/prefecture/:prefecture/label/:label/list'
    },
    {
      // 都道府県x市区町村x絞り込みラベルの物件リスト ④
      source: '/prefecture-:prefecture(\\w+)/city-:city(\\d+)/label-:label(\\w+)/list',
      destination: '/prefecture/:prefecture/city/:city/label/:label/list'
    }
  ]
}

module.exports = {
  rewrites
}
```
このように正規表現織り交ぜることで、`/city-aaa`のようなルールに沿わないアクセスが、自動的に404ページに転送される。

ディレクトリ構成
```
/pages
└── prefecture
    └── [prefecture]
        ├── city
        │   └── [city]
        │       ├── label
        │       │   └── [label]
        │       │       └── list.tsx ④
        │       └── list.tsx ③
        ├── label
        │   └── [label]
        │       └── list.tsx ②
        └── list.tsx ①
```

```tsx
// pages/prefecture/[prefecture]/city/[city]/label/[label]/list.tsx
export const getStaticProps: GetStaticProps<
  PageProps,
  {
    prefecture: string
    city: string
    label: string
  }
> = async ({ params }) => {
  // /prefecture-tokyo/city-123/label-one_room/list にアクセスした場合
  // params: { prefecture: 'tokyo', city: '123', label: 'one_room' }
  // のように格納されている
}
```

### 注意点
- `trailingSlash: true` にしている場合、`source`の末尾にもスラッシュが必要である。
	- `/list(/{0,1})`このように書いて、末尾スラッシュ有り無し両方に対応しても良い
- id など数字のみが入るようにパスを制御できるが、page側では、`string`に変換されている
- 利用しているデプロイプラットフォームによって、正規表現の解釈が若干異なるため、ローカルでは動いたのにデプロイしたら挙動が異なるということがまれにある。
	- 正規表現部分はあまり複雑にしないほうが良い
	- [serverless-next](https://github.com/serverless-nextjs/serverless-next.js)を利用する場合、内部で利用している`path-to-regexp`が`[a-z]`のようなブラケット型の正規表現を解釈できない
	- Vercelでは、ブラケット型の正規表現にも対応している

## Addition
もとのダイナミックルーティングによるパスも生きているので、そのままだと `/prefecture/tokyo/city/123/label/one_room/list` のようなパスでのアクセスも可能になってしまう。
SEOを意識したサイトを作る場合、評価分散を避けるため、redirectsルールを使用してダイナミックルーティングによるパスを無効にしてやる
```js
// next.config.js
const redirects = async () => {
  return [
    {
      // 都道府県の全物件リスト
      source: '/prefecture/:prefecture(\\w+)/list',
      destination: '/prefecture-:prefecture/list',
      statusCode: 301 // 省略した場合、ステータスコードは308
    },
    {
      // 都道府県x市区町村内の物件リスト
      source: '/prefecture/:prefecture(\\w+)/city/:city(\\d+)/list',
      destination: '/prefecture-:prefecture/city-:city/list',
      statusCode: 301
    },
    {
      // 都道府県x絞り込みラベルの物件リスト
      source: '/prefecture/:prefecture(\\w+)/label/:label(\\w+)/list',
      destination: '/prefecture-:prefecture/label-:label/list',
      statusCode: 301
    },
    {
      // 都道府県x市区町村x絞り込みラベルの物件リスト
      source: '/prefecture/:prefecture(\\w+)/city/:city(\\d+)/label/:label(\\w+)/list',
      destination: '/prefecture-:prefecture/city-:city/label-:label/list',
      statusCode: 301
    }
  ]
}

module.exports = {
  rewrites,
  redirects
}
```

## Results

https://rewrites-sample.vercel.app/prefecture-tokyo/city-123/list

https://rewrites-sample.vercel.app/prefecture-tokyo/city-123/label-one_room/list

形式の沿わない city (city-aaa) は 404になる
https://rewrites-sample.vercel.app/prefecture-tokyo/city-aaa/list

直接 ダイナミックルーティングにアクセスを試みると、301転送される
https://rewrites-sample.vercel.app/prefecture/tokyo/city/123/list

今回のコードはこちらのリポジトリを参照
https://github.com/aiji42/rewrites-sample