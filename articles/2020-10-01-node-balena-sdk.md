---
title: nodeからBalenaSDKをサーバーで使うときはdataDirectoryを設定する
emoji: 🐋
type: tech
topics:
  - balena
  - nodejs
published: true
---

BalenaのSDKを利用した際、ローカルで実行するとローカルの`balena-cli`のトークンを勝手に使ってしまって困ったのでメモ。

結論としてはこれを防ぐには下記のように`dataDirectory`を設定するしかなさそう

```ts

const sdk = getSdk({
  dataDirectory: process.env.BALENA_DATA_DIR || "/tmp",
})

await sdk.auth.loginWithToken(process.env.BALENA_API_KEY)
```

`datDirectory`は書き込めるディレクトリであれば何でも良いが、多くのlinux環境で利用できる`/tmp`をここでは利用した。

どうもこれはbalena-authの仕様的にそうやっているっぽかった。

https://github.com/balena-io-modules/balena-auth/blob/7158f624cd318f3c3015e787a4aafe3d1a546f32/lib/auth.ts#L34-L40
