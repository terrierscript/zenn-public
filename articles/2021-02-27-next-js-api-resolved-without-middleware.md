---
title: next.jsでAPI resolved without sending a response...のエラーが出るとき
emoji: 🚰
type: tech
topics:
  - nextjs
  - javascript
  - typescript
published: true
---

next.jsで時折下記のようなエラーが出ることがある

```
API resolved without sending a response for /api/hello, 
this may result in stalled requests.
```

これは主に`res.end()`を忘れたり`res.json()`を使わず、レスポンスの終了処理をしてないときに起きる

```ts
export default async (req, res) => {
  const { status } = await axios.get("https://example.com")
  res.status(status)
  // res.status(status).end() とすべき
}
```

上記の例の場合、警告が出るだけでなくCloud Runなどの場合は不必要に料金が発生するので注意しなければならない。

## Middlewareの書き方で起きるケース

これは例えばmiddlewareの書き方で起きることもある。
例えば下記のような書き方をした場合に起きる

```ts
const exampleMiddleware = (handler: NextApiHandler) => {
  return (req, res) => {
    console.log("this is example middleware!")
    handler(req, res)
  }
}

async function someHandler(req, res) {
  const { status } = await axios.get("https://example.com")

  res.status(status).end()
}

export default exampleMiddleware(someHandler)
```

一見`.end()`もしているのが、`exampleMiddleware`に問題がある。

### 解決法

`return`でhandlerを返してあげる。

```ts
const exampleMiddleware = (handler: NextApiHandler) => {
  return (req, res) => {
    console.log("this is example middleware!")
    //  handler(req, res) <- これだと警告が出る
    return handler(req, res)
  }
}
```

またはasync/awaitの形式で記述する

```ts
const exampleMiddleware = (handler: NextApiHandler) => {
  return async (req, res) => {
    console.log("this is example middleware!")
    await handler(req, res)
  }
}
```
---
参考:
* https://github.com/vercel/next.js/issues/10439
* https://github.com/vercel/next.js/blob/004ad62d6b69534bf7cc0df13cc174167e616aa8/packages/next/next-server/server/api-utils.ts#L89-L97