---
title: next-authを複数のプロジェクトで利用してるときにローカル環境が奪い合いになる問題に向き合う
emoji: 🏘️
type: tech
topics: [typescript, nextjs, nextauth]
published: true
---

next-authを複数のプロジェクトで利用を利用する場合、ローカルで同時に実行すると同一のCookieを参照するため、片方でログインすると片方がログアウトされるという状態が起きてしまう。

別ブラウザを利用するなどの対処法もあるが、いちいち面倒なので、対処を考える

# Cookie名を変更する

この問題は同名のcookieを上書きしあってしまうことなので、cookie名を変更すれば良い。

`next-auth`のcookie自体は、optionで変更が可能になっているのでこれを利用していく

```ts
// pages/api/auth/[...nextauth].js

const cookieOptions = ... // この部分については後述
export const authOptions = {
  // Configure one or more authentication providers
  providers: [
    GithubProvider({
      clientId: process.env.GITHUB_ID,
      clientSecret: process.env.GITHUB_SECRET,
    }),
    // ...add more providers here
  ],
  cookies: cookieOptions, 
}
export default NextAuth(authOptions)

```

`middleware`を利用している場合は、そちらの設定も変更する必要がある。

```ts
// middleware.ts
const auth = withAuth({
  cookies: cookieOptions
})
```

## Cookie設定を生成する

ここからCookieの設定を生成することを考える。
いくつか方法があるので、行儀の良い順に記載する

### 行儀の良いやりかた : Exampleをコピーしてくる
ドキュメントにある[Example](https://next-auth.js.org/configuration/options#example)からコピーして書き換える。

丁寧だが若干面倒。複数プロジェクトあると面倒なのと、アップデート時などに変更を来にする必要はある

### 程々に行儀の悪いやりかた : nameのみ定義する

nameのみ変更した状態で管理することでも目的は達成できる。

```ts
const prefix = `my-app`
// 本来は cookies: Partial<CookiesOptions> 
const cookies: any = {
  sessionToken: { name: `${prefix}-auth.session-token` },
  callbackUrl: { name: `${prefix}-next-auth.callback-url`} ,
  csrfToken: { name: `${prefix}-next-auth.csrf-token` },
  pkceCodeVerifier: { name: `${prefix}-next-auth.pkce.code_verifier` },
  state: { name: `${prefix}-next-auth.state` },
  nonce: { name: `${prefix}-next-auth.nonce` }
}
```

この方法は内部実装で設定を[deep merge](https://github.com/nextauthjs/next-auth/blob/af246b79ed8da5603ae8a6ef4c1f69ecb27e54f5/packages/core/src/lib/init.ts#L111)していることをアテにしているので、割と行儀が悪い

### しっかり行儀の悪いやりかた : 内部実装のdefaultCookiesを引っ張ってくる

内部でデフォルトのcookieを生成している`defaultCookies`を呼び出して書き換える方法もある。
この関数、VS Codeの補完などでは下記のように出てくるので一見使えそうなのだが、実際にはimportできていない。

```ts
import { defaultCookies } from "next-auth/core/lib/cookie"

// Module not found: Package path ./core/lib/cookie is not exported from package 
```

* https://github.com/nextauthjs/next-auth/issues/4709
* https://github.com/nextauthjs/next-auth/discussions/6043


ただ、呼び出す方法が無いわけでもなく、[このissueコメント](https://github.com/nextauthjs/next-auth/issues/6649#issuecomment-1753491597)を参考に、`node_modules`を経由して呼び出すことはできる。

```tsx
// lib/next-auth/cookie.ts
import { defaultCookies } from '../../../node_modules/next-auth/core/lib/cookie'

const prefix = "my-app"
export const getCookie = () => {
  if (process.env.NODE_ENV !== 'development') {
    return {}
  }
  return Object.fromEntries(
    Object.entries(defaultCookies(true)).map(([key, value]) => {
      return [key, {
        ...value,
        name: `${prefix}_${value.name}`
      }]
    })
  )
}

```

これを利用すると、行儀が悪いものの目的のcookieを生成することができる。
