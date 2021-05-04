---
title: 社内向けの管理画面の認証にSlack認証を使うと便利だった
emoji: 🔐
type: idea
topics:
  - slack
  - nextjs
  - nextauth
published: true
---

社内の管理ツール、ちょっとした便利ツールだと独立してサクッと出来てしまったりするのだが、意外と認証が面倒だったりする。

しかし「Basic認証はちょっと弱いんだけどDB管理まではしたくないし…」みたいなレベルがよくある。

社内にG Suiteが導入されている場合はだいたいGoogle認証やFireAuthをよく使っていたのだが、下記のような問題もあった

* FireAuthで管理する（Email/Password方式やGoogleAuthProvider認証など）
  * -> 既存でユーザー向けに提供している場合はユーザーと管理者が混在してしまう
  * -> 一方で新しくfirebaseプロジェクトを作るのも管理が面倒になる
* Google認証を素で使う
  * GoogleのAuthもだいたいあるが、割と細かいところで面倒くささがある。
  * credentialもファイル管理が必要で気軽にenvに入れづらい

## そこでSlack認証を使ってみる

そんな折、[next-auth](https://next-auth.js.org)を使おうと見ていたら[Slack Provider](https://next-auth.js.org/providers/slack)があるのを見つけた。

比較的多くの企業であればSlackを使っている上、基本的に各社員に配布され、退職すればrevokeされるのでこれは管理ツールに使うにはとてもちょうどよいケースが多そうだ。

ちなみにNextAuthのProviderはかなり豊富なので、開発者のみで良ければ[GitHub](https://next-auth.js.org/providers/github)や[GitLab](https://next-auth.js.org/providers/gitlab)、あとは[Discord](https://next-auth.js.org/providers/discord)もあったりするので導入している場合はそれでも良いだろう

また、今回はnext.jsを中心に話しているが、他のフレームワーク等でもslack認証が存在する場合がある

* [Laravel/Socialite/slack](https://github.com/SocialiteProviders/Providers/tree/master/src/Slack)
  * なぜかドキュメントページは見当たらなかった
* [Rails/omniauth-slack](https://github.com/kmrshntr/omniauth-slack)
  * かなり前に更新が止まっているようなので、動いているのか未検証

## next-authでやってみる

### Example
下記に配置した。
https://github.com/terrierscript/example-next-auth-slack/

かなり導入も簡単なので[Example](https://next-auth.js.org/getting-started/example)通りやっただけのものになる。

### Slack側の手順

Slackのほうは下記のような手順が必要になる

1. https://api.slack.com/apps で「Create New App」
2. App Credentialsの「ClientId」と「ClientSecret」をコピーする
3. Features -> OAuth & Permissions の項目を選び、Callback URLを設定する。だいたい下記のようになるだろう。
    * ローカル向けには`http://localhost:3000/api/auth/callback/slack`
    * 本番なら `https://some-your-domain/api/auth/callback/slack`
4. 2でコピーしたClientIdとClientSecretを`.env`に転記する。なお、本番向けにデプロイする場合は`NEXTAUTH_URL=https://some-your-domain/`のような記述も必要だろう

こんな具合でおそらく動くだろう。

### 残課題
* 共有ワークスペースのときにどうなるかが未検証。
  * おそらく[callback設定のsignIn](https://next-auth.js.org/configuration/callbacks#sign-in-callback)で`teamId`か`workspaceId`が正しくない場合失敗させる、という処理が必要そうだが、共有ワークスペースを試せたことが無いので不明…