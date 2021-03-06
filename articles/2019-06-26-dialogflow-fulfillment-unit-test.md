---
title: FirebaseにアップされたDialogflowのFulfillmentをUnit Testする
emoji: 🔥
type: tech
topics:
  - dialogflow
  - mocha
  - javascript
  - firebase
published: true
terrier_date: '2019-06-26T13:22:41.375Z'
terrier_export_from: >-
  https://blog.terrier.dev/blog/2019/20190626222241-dialogflow-fulfillment-unit-test
---

> 記事作成日: 2019/06/26

Dialogflowで利用するfirebase functionの関数をテストしたくで調べた。

## 準備編
こんな感じでまず関数を準備する

```js
const functions = require("firebase-functions")
const { WebhookClient } = require("dialogflow-fulfillment")

exports.dialogflowFirebaseFulfillment = functions.https.onRequest(
  (request, response) => {
    const agent = new WebhookClient({ request, response })
    const intentMap = new Map()
    function hello(agent) {
      const { name } = agent.parameters
      agent.add(`ハロー! ${name}`)
    }

    intentMap.set("hello", hello)
    agent.handleRequest(intentMap)
  }
)
```

## テストを書く

こんな感じで素直にテストを書く。
firebase-functions-testは使えないので、素直にreqとreqを偽造して渡す。
https://firebase.google.com/docs/functions/unit-testing?hl=ja

もうちょっと丁寧にやるなら[node-mocks-http](https://github.com/howardabrams/node-mocks-http)を使う手もあったが、今回はそこまでやらなかった。

```js
const assert = require("assert")
const { dialogflowFirebaseFulfillment } = require("../simple.js")

describe("function", () => {
  it("janken", (done) => {
    const req = {
      body: {
        queryResult: {
          queryText: "ハロー",
          parameters: {
            name: "太郎"
          },
          intent: {
            displayName: "hello"
          }
        }
      }
    }

    const res = {
      json: (j) => {
        console.log(j)
        const expect = { fulfillmentText: "ハロー! 太郎", outputContexts: [] }
        assert.deepEqual(expect, j)
        done()
      }
    }

    dialogflowFirebaseFulfillment(req, res)
  })
})
```

## Requestをどう組み立てるか？

ちなみに、気づきづらいが、右サイドで入力テストが出来る。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">このTry it now入力できたのか！分かりづらい！ <a href="https://t.co/BaRlsuYO74">pic.twitter.com/BaRlsuYO74</a></p>&mdash; @terrierscript <a href="https://twitter.com/terrierscript/status/1143808200359084033?ref_src=twsrc%5Etfw">June 26, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

そして「DIAGNOSTIC INFO」をクリックし、「FULLFILMENT REQUEST」のタブを開くと、RequestとしてAPIに入ってくるjsonが取得出来るので、これを元にデータをつくると良い

<blockquote class="twitter-tweet" data-conversation="none" data-lang="ja"><p lang="ja" dir="ltr">しかし結果がすげえわかりやすい・・・ <a href="https://t.co/hTxYPZJueY">pic.twitter.com/hTxYPZJueY</a></p>&mdash; (@terrierscript) <a href="https://twitter.com/terrierscript/status/1143809073227264000?ref_src=twsrc%5Etfw">2019年6月26日</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
