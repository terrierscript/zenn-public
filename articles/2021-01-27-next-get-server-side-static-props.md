---
title: next-authとgetServerSide(Static)Props使うときの__NEXT_DATA__に注意
emoji: 🚦
type: tech
topics:
  - nextauth
  - nextjs
  - javascript
published: true
---

next-authなどでクライアントログインを仕掛けていてgetServerSidePropsやgetStaticPropsでちゃんと認証かけてないとだだ漏れするので気をつけましょうねという話。

通常のサーバーサイド開発に慣れていればあまりハマるようなことではないのだが、知らないで使うと危ないので記述しておく

## 例示

例えば下記のようなコードで考えてみる。
だいたいログイン判定はクライアントなら`_app.js`にて書くだろう。

```jsx
// _app.jsx
import { signIn, useSession } from "next-auth/client"

export const NeedLogin = ({children}) => {
  const [session, loading] = useSession()
  if (loading) {
    return null
  }
  if (!session ) {
    return <div>
      Not signed in <br />
      <button onClick={signIn}>Sign in</button>
    </div>
  }
  return children
}

function MyApp({ Component, pageProps }) {
  return (
    <Provider options={{ clientMaxAge: 60 }} session={pageProps.session}>
      <NeedLogin>
        <Component {...pageProps} />
      </NeedLogin>
    </Provider>
  )
}

export default MyApp
```

そしてページ側はこのような具合に書いてみる

```jsx
// pages/somePage.jsx

import React from "react"
import { useSession } from "next-auth/client"
import { getSomeSecretData } from "../lib/getSomeData"


export default function Page({data}) {
  const [, loading] = useSession()
  if (loading) {
    return null
  }
  return (
    <div>
      Home
      <pre>Need Login: {JSON.stringify(data)}</pre>
    </div>
  )
}

export async function getServerSideProps() {
  const data = getSomeSecretData() // 例えばこれがログイン後にのみ表示させたいデータだとする
  return {
    props: { data },
  }
}
```

そうするとログイン後、ログイン前でこんな感じで表示されるはずだ

|ログイン前|ログイン後|
|--|--|
|![ログイン前](https://user-images.githubusercontent.com/13282103/105956881-ca0e0780-60bb-11eb-8019-696fd890e867.png)|![ログイン後](https://user-images.githubusercontent.com/13282103/105956963-e611a900-60bb-11eb-94e4-c485ef16954e.png)|

これにてめでたしめでたし。ではない。

ログイン前のページでinspectorをしてみよう

![](https://user-images.githubusercontent.com/13282103/105956901-cda18e80-60bb-11eb-8f65-6d07add6ebf8.png)

```html
<script id="__NEXT_DATA__" type="application/json">{"props":{"pageProps":{"data":["This","Data","Need","Login"]},"__N_SSP":true},"page":"/","query":{},"buildId":"development","isFallback":false,"gssp":true}</script>
```

このように`__NEXT_DATA__`としてgetServerSidePropsの値は表示されてしまう

## 対応方法

`getServerSideProps`で行う場合は[`getSession`](https://next-auth.js.org/getting-started/client#getsession)でチェックを入れましょう。

```js
import { useSession,getSession } from "next-auth/client"

export async function getServerSideProps(context) {
  const session = await getSession(context) // awaitを忘れずに
  if (!session) {
    return { props: {} }
  }
  
  const data = getSomeSecretData()
  return {
    props: { data },
  }
}
```

こうすれば`__NEXT_DATA__`にも表示されません。 [1]
[1]: `await`を忘れると`session`がPromiseになってしまい`if(!session)`を通過してしまうので注意

![OKなパターン](https://user-images.githubusercontent.com/13282103/105958973-a7c9b900-60be-11eb-945f-83a1e634da6e.png)

```html
<script id="__NEXT_DATA__" type="application/json">{"props":{"pageProps":{},"__N_SSP":true},"page":"/","query":{},"buildId":"development","isFallback":false,"gssp":true}</script>
```

また、`getStatciProps`の場合はリクエストなどの`context`が取得できないためこのままではできません。もともと`getStatciProps`はログインが必要などのユースケースは含まれてないため、どうしても`getStaticProps`を利用する必要がある場合はAPI経由などで取得をするのが良いでしょう。

APIの場合であれば上記同様`getSession`が利用できます