---
title: tRPCとnext.jsの組み合わせでwithTRPCをしないで利用したい話とtRPC便利だったとこメモ
emoji: 🐯
type: tech
topics:
  - trpc
  - typescript
  - nextjs
published: true
---

next.jsのAPI周りは気軽に使えて便利なのだが、型周りを考えるとちょこちょこ事故が起きやすい箇所になる。
[tRPC](https://trpc.io/)を利用するとかなり堅牢なAPIを作ることができる

## ざっくりtRPCとnext.jsにおける利用について

軽くtRPCとnext.jsとの組み合わせについてかいつまむと、下記のような形になる

* 名前の通りTypeScriptを利用したRPCを行うためのライブラリ
* エンドポイントを一つに固定してクライアントでも利用することで型を共有できるもの
* next対応については[Usage with Next.js](https://trpc.io/docs/nextjs)としてドキュメント化されてるぐらいがっちりサポートされている
* next.jsの普通のAPIのルーティングとは異なるため、`/api/trpc/[trpc]`のような形でエンドポイントを一つに絞って利用する
* Contextのような機能もあって、複数のAPIの共通処理を一括管理できるところは嬉しい

## もうちょっとライトに使いたい

公式のドキュメントとして提供されているnext.jsとの組み合わせだと、下記のような点を個人的にネックに感じた

* コンポーネントを`withTRPC`でHoCすることでSSRを対応しているが、若干個人的にほしい機能に比べると過剰
  * クライアント認証を挟むような場合だと若干過剰
* SWR派なので、react-queryを利用する部分がちょっと悩ましい
  * react-queryを使うにしても一つ古い`v3`系にされているので、ちょっと気が進まない
* `middleware`やContextなどの処理が違う複数のエンドポイントを作りたい要件だと同様課題がある

## 対処：`withTRPC`は利用せず、VanillaなClientと組み合わせる

tRPCはフレームワークを問わないライブラリで、素のTypeScript向けにも利用できるようになっているので、クライアント部分はこれを利用すれば依存度が低く、自由度が高く使えそうだった。

* https://trpc.io/docs/vanilla

今回は上記のVanillaなクライアントを利用して、下記の部分をカスタマイズするようなものを考えたい

* 複数のエンドポイントを作る
* `withTRPC`を使わない
* SWRと組み合わせる

### サーバー側

複数のエンドポイントを分けたいので、下記のようにエンドポイントを複数作る。

* `/pages/api/user/trpc/[trpc].ts`
* `/pages/api/admin/trpc/[trpc].ts`

中身は普通のnext.js用のものを利用すれば良い。ほとんど[ドキュメント通り](https://trpc.io/docs/nextjs#3-create-a-trpc-router)

```ts
import * as trpc from '@trpc/server'
import * as trpcNext from '@trpc/server/adapters/next'
import { z } from 'zod'

export const adminRouter = trpc
  .router()
  .query('hello', {
    input: z.object({ text: z.string().nullish() }).nullish(),
    resolve({ input }) {
      return {
        greeting: `hello ${input?.text ?? 'world'}`,
      }
    },
  })

// ルーターの名前は変えておく
export type AdminAppRouter = typeof adminRouter

// export API handler
export default trpcNext.createNextApiHandler({
  router: adminRouter,
  createContext: () => null,
})
```
`/pages/api/user/trpc/[trpc].ts` もほぼ同様となるので省略する

### クライアント側

素直に`createTRPCClient`が利用できるので、それぞれ`url`でエンドポイントを切り替えたものを用意する。
`useMemo`を利用しているが、用途によっては特にhooks化する必要もないかもしれない

```tsx
const useAdminTrpc = () => {
  const client = useMemo(() => {
    return createTRPCClient<AdminAppRouter>({
      url: '/api/admin/trpc',
    })
  }, [])
  return client
}
const useUserTrpc = () => {
  const client = useMemo(() => {
    return createTRPCClient<UserAppRouter>({
      url: '/api/user/trpc',
    })
  }, [])
  return client
}
```

コンポーネント側では下記のように利用する。

```tsx
const Greeting = () => {
  const trpc = useAdminTrpc()
  const { data } = useSWR("hello", () => {
    return trpc.query("hello")
  })
  return <Box>
    <Box>{data?.greeting}</Box>
  </Box>
}
```

SWRと組み合わせる部分はキーが重複したりしそうな部分もあるので、ちょっとここは注意が必要
本当はもう少し頑張れば`useSWRTrpcQuery`みたいなのも作れるが、そこまでほしい場合は[`trpc-swr`](https://github.com/sachinraja/trpc-swr)の利用を検討するのが良いかもしれない

## その他：　tRPCの良かったとこ

tRPCを一通り触って便利だった部分メモ書き

### マージ機能
* https://trpc.io/docs/merging-routers

Routerの部分が太ってしまいそうだなというのが結構懸念だったが、mergeする機能があるのでこれは便利そうだった。

```ts
const appRouter = createRouter()
  .merge('user.', users) 
  .merge('post.', posts)
```

### Infer typeとContext
* https://trpc.io/docs/infer-types
* https://trpc.io/docs/context

各種Infer系が揃っているので、これも型を再定義しないのは楽に感じた。

Context周りも`inferAsyncReturnType`などを利用すれば良いようだた。
ざっくり組み合わせると下記のような具合に共通化できてよかった

```ts
const createUserAppContext = (opts?: trpcNext.CreateNextContextOptions) => {
  // 本来は認証など色々共通処理を行う
  return {
    foo: "baz"
  }
}

type UserAppContext = trpc.inferAsyncReturnType<typeof createUserAppContext>

const createUserAppRouter = () => {
  return trpc.router<UserAppContext>()
}

export const userRouter = createUserAppRouter()
  .query( .... )

// export type definition of API
export type UserAppRouter = typeof userRouter

// export API handler
export default trpcNext.createNextApiHandler({
  router: userRouter,
  createContext: createUserAppContext
})
```

### trnasform周り

* https://trpc.io/docs/data-transformers

Prismaとnext.jsを組み合わせる際、Date型が引き継げなくて`superjson`を挟む必要が出てきたりするのだが、これも`transform`というオプションを使えば良い。クライアント側、サーバー側どちらも設定するのを忘れずに

```ts
// サーバー側
export const appRouter = trpc.router()
  .transformer(superjson)
```

```tsx
// client側
const useUserTrpc = () => {
  const client = useMemo(() => {
    return createTRPCClient<UserAppRouter>({
      url: '/api/user/trpc',
      transformer: superjson,
    })
  }, [])
  return client
}
```
