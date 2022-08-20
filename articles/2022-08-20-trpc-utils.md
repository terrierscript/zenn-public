---
title: tRPCの便利だった機能
emoji: 🐅
type: tech
topics:
  - trpc
  - typescript
  - nextjs
published: true
---

[nextと組み合わせる](https://zenn.dev/terrierscript/articles/2022-08-18-trpc-nextjs-without-hoc)のをやりながら見つけた便利だった部分についての列挙

### マージ機能
* https://trpc.io/docs/merging-routers

Routerの部分が肥大化してしまいそうだなというのが結構懸念だったが、mergeする機能があるのでこれは便利そうだった。

```ts
const appRouter = createRouter()
  .merge('user.', users) 
  .merge('post.', posts)
```

### Infering Types
* https://trpc.io/docs/infer-types

Routerなどの各種定義からTypeScriptの型を取り出す機能。
ちょっと記述が面倒なところはあるが、`AppRouter`の返り値や入力値が抽出できるので、一通り覚えておくと便利。
（とはいえinputについては`zod`のinferを利用する方がスッキリするような気もする）

### Context

それぞれのrouterの処理の事前にContextを取得できるのが便利。

* https://trpc.io/docs/context

例えばCookieからAuthしたりなどができる。

inferについても、`inferAsyncReturnType`などを利用すれば良いようだた。
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

### transform機能

* https://trpc.io/docs/data-transformers

Prismaとnext.jsを組み合わせる際、Date型が引き継げなくて`superjson`を挟む必要が出てきたりするのだが、これも`transform`というオプションを使えばバッチリできた。

クライアント側、サーバー側どちらも設定しないと片方がズレて適切に取り出せないので注意

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
      url: '/api/trpc',
      transformer: superjson,
    })
  }, [])
  return client
}
```

### サーバー側のエラーハンドリング

* https://trpc.io/docs/error-handling#handling-errors

クライアントにはエラーが返ってくるが、サーバー側でもエラーをログに出すなどしたい場合は、明示的に`onError`を指定しておく必要があるらしいようだった

```ts
export default trpcNext.createNextApiHandler({
  router: userRouter,
  createContext: createUserAppContext
  onError({ error, type, path, input, ctx, req }) {
    console.error(error);
  }
})
```

もしくは[`middleware](https://trpc.io/docs/middlewares)も用意されているので、こちらでエラーを出力するのでも良い可能性がある

### ヘッダ送信

* https://trpc.io/docs/header

Clinet側からHeaderを送るには`headers()`という関数で設定可能。
上記ドキュメントでは`withTRPC`で記述されているが、vanillaな`createTRPCClient`からでも可能

```ts
const useAuthorizedTrpc = (authToken: string) => {
  return createTRPCClient<AppRouter>({
    url: '/api/trpc',
    headers() {
      return {
        Authorization: authToken
      }
    }
  })
}
```

CookieやCORSなどfetch周りをいじる必要があるなら[fetch](https://trpc.io/docs/cors)関数自体を変更することで対処できる模様

### その他

まだあまりちゃんと触れてないがそれ以外の便利そうな部分

* middleware
  * https://trpc.io/docs/middlewares
  * よくあるミドルウェア機能。contextと
* メタデータ機能
  * https://trpc.io/docs/metadata
  * それぞれのハンドラでメタデータを設定し、middlewareなどで
* Output Validation
  * https://trpc.io/docs/output-validation
  * 出力についてバリデーションする機能。インターフェースを先に決めたい場合などには良いかも
* Request Batch
  * https://trpc.io/docs/links
  * tRPCの通信はバッチ化される。都合が悪い場合は設定でdisableにもできる
  * linksの設定などを使うと更に通信周りをカスタマイズできる模様
* Subscription
  * https://trpc.io/docs/subscriptions
  * Websocketっぽい通信もできるっぽい（Vercelやサーバレスに乗っけてるとできないので使い所難しい）

## まだ未解決な事

* クライアントサイドのコードからのコードジャンプがちょっとめんどい
* Routerの型だけ抽出できたらプロジェクト間で共有するとかできて嬉しいなと思ったが、そういうのはなさそう
* inferは充実しているのだが、router側のハンドラー定義を細かく切り出そうとするとちょっと面倒
  * `CreateProcedureOptions`など`Procedure`系が定義はされているが、外部で利用できるような感じにはできてなさそう
  