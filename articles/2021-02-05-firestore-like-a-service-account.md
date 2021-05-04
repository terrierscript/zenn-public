---
title: firestoreにサービスアカウントっぽいアカウントで操作する
emoji: 🌚
type: idea
topics:
  - firestore
  - fireauth
  - firebase
  - javascript
published: true
---

Firestoreは概ね便利だが、厄介と感じる部分の一つに「`firebase-admin`から利用するとrulesを無視して問答無用でフルアクセスできてしまう」というのがある。

そこでサービスアカウントの概念よろしくFireauthでロボットアカウントを生成し、それを使ってみてはどうか？というアイディアを思いついたので覚書

:::message alert
なお、これらを筆者は本番運用したことは無いので、あくまでもPoCとお考えください
:::

前準備として、ロボットアカウントとなるIDをコンソールから作成する。
![](https://storage.googleapis.com/zenn-user-upload/oa3ozy3s1usqq485sg5acb4pknpg)

あとは下記のようなコードでルールを守りつつサーバーサイドで動かせるアカウントが出来る。

```js
const firebaseAdmin = require("firebase-admin")
const firebaseClient = require("firebase")
require("firebase/firestore");

const serviceAccount = require( "./YOUR-SERVICE-ACCOUNT.json")

const firebaseConfig = {
  // YOUR FIREBASE CLIENT CONFIG
}

// コンソールで作成したIDを貼り付け
const serviceAccountUserId = "YOUR_SERVICE_ACCOUNT_USER_ID"

const main = async () => {
  firebaseClient.initializeApp(firebaseConfig)

  firebaseAdmin.initializeApp({
    credential: firebaseAdmin.credential.cert(serviceAccount)
  })
  // adminを経由し、ユーザーのカスタムトークンを作成する
  const customToken = await firebaseAdmin.auth().createCustomToken(serviceAccountUserId)
  
  // sign-inする。これをやらないとちゃんとコケてくれる
  await firebaseClient.auth().signInWithCustomToken(customToken)
  
  const db = firebaseClient.firestore()

  // ↓ruleはこんな感じ
  // match /auth/{someobj} {
  // 	allow read, write: if request.auth != null;
  // }
  const result = await db.collection("/auth").add({
    foo: "baz",
  }).doc
  console.log(result)
  // ↓ ruleに基づいて書き込めないときはfailed
  // const resultFailed = await db.collection("/auth-cant-write").add({
  //   foo: "baz",
  // }).doc
  // console.log(resultFailed)
}
main()
```

おそらく`createCustomToken`を利用すれば特定のユーザーに成り代わって書き込むということも出来そうだが、こちらは試してない
（例えばfunctionsから特定のユーザーアカウントとして書き込みたいなどの要件の場合使えそうだが、そういう例をあまり見ない？探せばあるかも？）