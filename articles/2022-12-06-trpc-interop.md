---
title: tRPCのv9からv10にmigrateするinteropを使うときのclientのハマりどころ
emoji: 🛺
type: tech
topics:
  - trpc
  - typescript
  - javascript
published: true
---

tRPCがv9からv10で大幅な変更があった。
幸いにも`interop()`という互換機能があり、[マイグレーションガイド](https://trpc.io/docs/migrate-from-v9-to-v10)も十分にあるのでほとんど書き換えずに処理できる。

ただ一点、[clientの書き方](https://trpc.io/docs/migrate-from-v9-to-v10#http-specific-options-moved-from-trpcclient-to-links)は若干混乱があり、ハマりどころが存在していたのでその点についての覚書き

## v9の記法のまま利用したい場合は、createTRPCProxyClientではなくcreateTRPCClientを変更する

clientの利用方法として、v9では下記のように記述する。

```ts
const getTrpcClient = () => {
  return createTRPCClient<AppRouter>({
    url: `/api/trpc/`
  })
}
```

利用側はこのような感じで利用する

```ts
const client = getTrpcClient()
client.query("greeting", "tom")
```

マイグレーションガイドでは、`useTrpcProxyClient`を利用するように案内されている

```ts
const getTrpcProxyClient = () => {
  return createTRPCProxyClient<AppRouterV10>({
    links: [
      httpBatchLink({
        url: `/api/trpc/`
      })
    ]
  })
}
```

しかしこの記述の場合、下記のようにclientの利用記法を変更しなければならない。

```ts
const client = getTrpcProxyClient()
client.greeting.query("tom")
```

また、`createTRPCProxyClient`を利用した場合、`interop()`したルーターは型補完が効かないようだった

### 解決法：createTRPCClientの引数を変更する

v9で`interop()`したルーターをそのままの記法で利用するには、`createTRPCClient`をそのまま使う必要がある。

```ts
const getTrpcClient = () => {
  return createTRPCClient<MergedAppRouter>({
    // url: `/api/trpc/`
    links: [
      httpBatchLink({
        url: `/api/trpc/`
      })
    ]
  })
}
```
URLのオプションは消え、`links`の形で`createTRPCProxyClient`と同様のパラメータを渡すと互換性を保った状態で動作し、型補完も効くようになる。
`createTRPCClient`はdeprecatedになっているのでVSCode上では取り消し線が出るが、一旦の互換処理としては十分だろう。