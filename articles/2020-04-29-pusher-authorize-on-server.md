---
title: Pusherのクライアントをサーバーサイドで自己認証する
emoji: 🐾
slug: 2020-04-29-pusher-authorize-on-server
type: tech
topics:
  - pusher
  - javascript
  - node
published: true
terrier_date: '2020-04-29T06:48:10.446Z'
terrier_export_from: https://blog.terrier.dev/blog/2020/20200429154810-pusher-authorize-on-server/
---


Pubsubサービスの[Pusher](https://pusher.com/)。

Pusherの`subscription`クライアントは基本的に[`pusher-js`](https://github.com/pusher/pusher-js)を利用して行う。
そしてこのクライアント自体は`node`環境にも対応している。

また、private/presenceチャンネルの場合、[`pusher-http-node`](https://github.com/pusher/pusher-http-node)を利用してauthorizeを通す必要がある。
これは基本的にサーバーを別途立てて`/pusher/auth`へPOSTするような作りになっている。

## 問題点
この場合、例えばEdgeデバイスなどのnode環境上でサブスクリプションするクライアントを作る時いくらか面倒が発生する。サーバー向けに[webhooks](https://pusher.com/docs/channels/server_api/webhooks)もあるのだが、これは一点に集約するためで、Edgeサーバー向きではない

## 解決策

`pusher-js` は実は[`authorizer`](https://github.com/pusher/pusher-js#authorizer-function)というオプションを持っており、これで自己認証するクライアントを作成できる（当然だが、PUSHER_SECRETを秘匿して扱えるのが前提になる

```js
const Pusher = require("pusher-js/node")
const PusherServer = require("pusher")
const uuid = require("uuid")


const pusherServer = new PusherServer({
  appId: PUSHER_APP_ID,
  key: PUSHER_KEY,
  secret: PUSHER_SECRET,
  cluster: "ap3",
  encrypted: true,
})
// Pusher.logToConsole = true
const pusher = new Pusher(PUSHER_KEY, {
  cluster: "ap3",
  authorizer: (channel) => {
    return {
      authorize: (socketId, callback) => {
        const auth = pusherServer.authenticate(socketId, channel.name, {
          user_id: `server-${uuid.v4()}`, // we require userId when presence channel
        })
        callback(false, auth)
      },
    }
  },
})
```
