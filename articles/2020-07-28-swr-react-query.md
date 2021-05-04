---
title: SWRとReact Queryどっち使おうか検討したメモ
emoji: 🎩
type: idea
topics:
  - swr
  - react
  - javascript
  - reactquery
published: true
terrier_export_from: "Notion"
---


SWRとReact Query（以下RQ）をどっち使おうか眺めてて、「だいたい出来ることは一緒のはずなのになんかSWRのほうが惹かれる」ってなったのでどのへんが起因だったかのメモ

### ファーストExampleに見る違い

[https://swr.vercel.app/#overview](https://swr.vercel.app/#overview)

```jsx
const { data, error } = useSWR('/api/user', fetcher)
```

[https://react-query.tanstack.com/docs/overview#enough-talk-show-me-some-code-already](https://react-query.tanstack.com/docs/overview#enough-talk-show-me-some-code-already)

```jsx
  const { isLoading, error, data } = useQuery('repoData', () =>
     fetch('https://api.github.com/repos/tannerlinsley/react-query').then(res =>
       res.json()
    )
 )
```

一見、ほとんど一緒じゃん、となるが、SWRは「キーをfetcheの実行するためのパラメータ」になっている。

一方、react-queryは「キーと実行APIは独立している」。これはイメージとしてReduxのActionなどのconstantのような印象を受ける

## APIの違い

### Core API

- RQ : [https://react-query.tanstack.com/docs/api](https://react-query.tanstack.com/docs/api)
- SWR [https://swr.vercel.app/docs/options#parameters](https://swr.vercel.app/docs/options#parameters)

これはかなり明確に違いが出ている。RQの方はかなり豊富に返り値を用意している一方、SWRはisLoadingのようなものすら無い（正直`isLoading`ぐらいほしい）

### Configuration

- RQ: [https://react-query.tanstack.com/docs/api#reactqueryconfigprovider](https://react-query.tanstack.com/docs/api#reactqueryconfigprovider)
- SWR: [https://swr.vercel.app/docs/global-configuration](https://swr.vercel.app/docs/global-configuration)

どっちもGlobal Configを持ってた

## key

- SWR: [https://swr.vercel.app/docs/arguments#multiple-arguments](https://swr.vercel.app/docs/arguments#multiple-arguments)
- RQ: [https://react-query.tanstack.com/docs/guides/queries#query-keys](https://react-query.tanstack.com/docs/guides/queries#query-keys)

Arrayを渡せるのはどちらも一緒。objectについては、RQの方は渡せるらしい。SWRの場合、「objectを渡すとキャッシュ効かないよ」となっている。

これは困るケースはあるかもしれないと思う一方、useEffectのdepsと同一のルールで、実に「Reactっぽい」という感じもする

### dependent query

- SWR:  [https://swr.vercel.app/docs/conditional-fetching](https://swr.vercel.app/docs/conditional-fetching)
- RQ: [https://react-query.tanstack.com/docs/guides/queries#dependent-queries](https://react-query.tanstack.com/docs/guides/queries#dependent-queries)

SWRの面白いのはkeyとしてfunctionを扱える部分。これによって複数のuseSWRが依存性を持つようなことが出来る。RQの方はこれをenableフラグで行っている。

[SWRの方は一見良くみえるが混乱や不必要なオーバーロードが起きる](https://twitter.com/tannerlinsley/status/1287897256838885376)とRQの作者から助言があったが、例示まで教えてもらえなかったので正直どのような実害の話をしてるのかよくわかってない

## fetcher / queryFn

- SWR:[https://swr.vercel.app/docs/data-fetching#fetch](https://swr.vercel.app/docs/data-fetching#fetch)
- RQ: [https://react-query.tanstack.com/docs/guides/queries#query-key-variables](https://react-query.tanstack.com/docs/guides/queries#query-key-variables)

どちらもkeyをfunctionを受け取ることは出来る。

ただ、RQの方は `fetchTodoById(key, todoId, { preview })`　となっておりやはり「keyは使わない・あくまでキャッシュのため」という思想を強く感じる。

SWRは keyをそのままURLとして使う機構によって`fetcher`を共通化しやすくなっている

## 言語

- SWR → フルTypeScript
- React Query → ~~型定義提供のみ~~
  - どうやらReqct Queryも[TypeScript化した模様](https://github.com/tannerlinsley/react-query/pull/767)

## Cacheの内部構造

- 基本どっちもContextを使ってる

## Business

- SWR: vercel社は割とnextを中心にやっていて、そりゃこの辺をやるよなという感じ。「vercelはproductionで使ってるよ」というのも多分vercel（旧now.sh）でいい感じにローディングされる機能に使ってるんじゃないかと思われる
- [https://learn.tanstack.com/p/react-query-essentials](https://learn.tanstack.com/p/react-query-essentials)
    - react-query自体の動画講座をやっている？

### DevTool

- RQ: ReactQueryDevtoolsがある
- SWR: なさげ
    - [https://github.com/vercel/swr/issues/291](https://github.com/vercel/swr/issues/291)

## 先人の比較とか

- [https://blog.logrocket.com/caching-clash-useswr-vs-react-query/](https://blog.logrocket.com/caching-clash-useswr-vs-react-query/)
- [https://matamatanot.hatenablog.com/entry/2020/05/05/204119](https://matamatanot.hatenablog.com/entry/2020/05/05/204119)
- [https://react-query.tanstack.com/docs/comparison](https://react-query.tanstack.com/docs/comparison)
- [https://github.com/vercel/swr/issues/127](https://github.com/vercel/swr/issues/127)
- [https://dev.to/justinramel/react-data-fetching-20ij](https://dev.to/justinramel/react-data-fetching-20ij)
