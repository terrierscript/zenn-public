---
title: 'nextjsの[[...props]]によるOptional catch all routesの使い道・利用例'
emoji: 📡
type: tech
topics:
  - nextjs
  - javascript
  - typescript
published: true
---


## `Optional catch all routes`

next.jsのDynamic Routingとして、[Optional catch all routes](https://nextjs.org/docs/routing/dynamic-routes#optional-catch-all-routes)というのがある。、

これはパスとしてで`[[...props]]`と記述するものになる。

つまりDynamic Routingは`[props]`, `[...props]`, `[[...props]]`と三種類の記述がある。
かなり似ていてややこしいが、下記のようにまとめられる。

|                        | `/foo` | `/foo/baz` | `/foo/baz/bar` |
|------------------------|--------|------------|----------------|
|`/foo/[props].jsx`      |   ❌   |     ⭕️      |      ❌        |
|`/foo/[...props].jsx`   |   ❌   |     ⭕️      |      ⭕️        |
|`/foo/[[...props]].jsx` |   ⭕️   |     ⭕️      |      ⭕️        |

`[[...props]]`はあまり利用例が多くはないが、例えば疑似ルーティングのような事が出来る。

## `Optional catch all routes`の利用例

next.jsにおいて複数のページでの`getServerSideProps`/`getStaticProps`やレイアウト等を共通化したい場合、通常は`getServerSideProps`/`getStaticProps`の処理を別途切り出して呼び出したり、レイアウトをコンポーネント化して切り出すのが一般的な手法となる。

この別解として、ルーティングで `[[...props]]`と記載するを利用した手法でこれを解決が出来る。

例えばわかりやすい例として、「同じデータを別な形式で表示をさせたい」のようなケースは想像しやすいだろう。

今回は「リスト表示」「タイル表示」「タイル表示（説明文付き）」のような切り替えを出来るものを想定してみる。

まずファイルとして`/dogs/[[path]].tsx`のようなファイルを作る。

`getServerSideProps`はこのような感じになる。

```tsx
/** /dogs/[[path]].tsx */

export const getServerSideProps: GetServerSideProps = async (req) => {
  const query = req.query
  const dogs = await getDogs() // 共通で利用するデータ取得

  return {
    props: {
      items: dogs,
      // string[]型に統一する。query.paths.join("/")してもいいかも
      paths: query.paths ?? [] 
    },
  }
}

```

ページの受け口はこんな具合になる。ここの実装はなんでもよが、概ね疑似ルータっぽくなることがわかる。

```tsx
const ViewRouter: FC<{ items: Item[], paths: string[] }> = ({ items, paths }) => {
  const path = paths.join("/")
  switch (path) {
    case "tile":
      return <TileView items={items} />
    case "tile/description":
      return <TileView items={items} withDescription />
    case "list":
    default:
      return <ListView items={items} />
  }
}

const Page = (props) => {
  return <Layout>
    <ViewRouter {...props} />
  </Layout>
}

```

`TileView`と`ListView`は単にコンポーネントの組み合わせなので割愛する。
気になる場合は[ソース](https://github.com/terrierscript/example-next-dynamic-rest/blob/main/src/pages/dogs/%5B%5B...paths%5D%5D.tsx)より確認いただきたい

## デモ

画像データには[dog.ceo](https://dog.ceo)を利用させてもらった

* https://example-next-dynamic-rest.vercel.app/dogs

![preview](https://user-images.githubusercontent.com/13282103/116550773-20484c00-a932-11eb-9017-f91afa5b562c.gif)

