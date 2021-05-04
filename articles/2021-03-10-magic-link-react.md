---
title: Magicを使ったマジックリンクログインをReactでの素振りサンプルと感想
emoji: 🪄
type: idea
topics:
  - react
  - typescript
published: true
---

[Magic](http://magic.link/)というマジックリング認証に特化したサービスを知ったので素振りしてみた記録。


サーバーサイドまで含め入ったサンプルは公式のチュートリアルにある
* https://magic.link/guides

逆にSPAのようなクライアントログインの最低限パターンのサンプルが見当たらなかったので、素振りを書いてみた

## サンプル

```tsx
import { Magic } from 'magic-sdk'
import { useState, useEffect } from 'react'

// Dashboardに表示されたpublishable keyを持ってくる。これは公開してよいキー
const MAGIC_LINK_PUBLIC = "pk_test_****************"

// hooksに処理をまとめる
const useMagicLogin = () => {
  const [isReady, setIsReady] = useState<boolean>(false)
  const [isLoggedIn, setIsLoggedIn] = useState<boolean>(false)
  const [loginEmail, setLoginEmail] = useState<string | null>(null)

  const getMetadata = async () => {
    const magic = new Magic(MAGIC_LINK_PUBLIC)
    const isLoggedIn = await magic.user.isLoggedIn()
    setIsLoggedIn(isLoggedIn)
    if (!isLoggedIn) {
      return
    }
    const metadata = await magic.user.getMetadata()
    setLoginEmail(metadata.email)
  }

  const checkLogin = async () => {
    setIsReady(false)
    await getMetadata()
    setIsReady(true)
  }
  const login = async (email: string) => {
    await new Magic(MAGIC_LINK_PUBLIC, { locale: "ja" })
      .auth
      .loginWithMagicLink({
        email: email,
        // showUI: false // 英語画面を避けたければこのオプションを付ける。
      })
    checkLogin()
  }
  const logout = async () => {
    const magic = new Magic(MAGIC_LINK_PUBLIC)
    await magic.user.logout()
    setLoginEmail(null)
    checkLogin()
  }
  useEffect(() => {
    checkLogin()
  }, [])
  return { isReady, isLoggedIn, logout, login, loginEmail, checkLogin }
}

// ログインフォーム
const LoginForm = ({ login }) => {
  const handleSubmit = async (event) => {
    event.preventDefault()
    const { elements } = event.target
    login(elements.email.value)
  }

  return <div>
    <form onSubmit={handleSubmit}>
    <label htmlFor="email">Email</label>
      <input name="email" type="email" />
      <button>Log in</button>
    </form>
  </div>
}

// ログイン後ページ
const AfterLogin = ({ logout, loginEmail}) => {
  return <div>
    <div>Welcome: {loginEmail}</div>
    <button onClick={() => { logout() }}>logout</button>
  </div>
}

// ルート部分
export const App = () => {
  const { isReady, isLoggedIn, logout,login, checkLogin,loginEmail } = useMagicLogin()
  if (!isReady) {
    return <div>loading...</div>
  }
  if (!isLoggedIn) {
    return <LoginForm login={login} />
  }
  return <AfterLogin logout={logout} loginEmail={loginEmail}/>
}
```

## 感想

### 良い点
* `pk_***` / `sk_****` というキーの区分けはstripeっぽく明快で扱いやすい
* ReactNative版があるの嬉しい。ほぼ上記と同じような実装で行ける
* マジックリンク実装に特化してるのは嬉しい部分
  * OAuth連携もあるっぽいが未検証
  * firebase認証より全体的に簡易に感じる
* もうすぐ新プライシングになるらしい（問い合わせると教えてくれる）

### 課題点
* ~~対応言語は英語のみっぽいので、社内ツールを超えてると国内ではまだ使うの難しそう~~
  * サポートに聞いてみたらどうやらできると教えてもらった
    * `new Magic(MAGIC_LINK_PUBLIC, { locale: "ja" })`
  * ドキュメント化はされてなさそう
  * source: https://github.com/magiclabs/magic-js/blob/e71599e157d32f8cc6d90ce278b2c1e717933c97/packages/provider/src/core/sdk.ts#L98
* nextとの組み合わせのexampleは正直ちょっとイマイチに見えた
  * https://vercel.com/blog/simple-auth-with-magic-link-and-nextjs
  * next-authが対応してくれたら嬉しくなりそう
* 名前が直球過ぎてググったり口頭説明するのめんどそう
* 仕様上7日しかしかセッションが持たず、延長もできないため、永続化するためにはMagicで認証した後に自前のセッションやjwtと交換しなければならない。
  * 実装例: https://github.com/magiclabs/example-react-express/tree/0abb4e8b889f38bcd53e6b9ded1142fc28a6229e
    * こちらはfirebaseの認証として行っている例: https://magic.link/posts/magic-firebase-integration
  * 永続的なセッション関連については検討されているっぽい
    * https://github.com/magiclabs/magic-js/issues/165
  * セッションとして利用できないなら意義薄いのでは？という面は感じるところもあるが、モバイルアプリの認証としてはわりかし手間少なくできるのは嬉しいかもしれない