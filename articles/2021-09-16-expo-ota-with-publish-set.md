---
title: 'expoのOTAを--no-publishとpublish:setを使ってなるべく安全に扱う'
emoji: 🚀
type: tech
topics:
  - expo
  - react
  - reactnative
  - typescript
  - javascript
published: true
---

Expoはビルドをサーバーで行ってくれたり、React nativeにまつわる様々な問題を対処してくれるとても良いツールだ。

ExpoにはOTAの機能もついており、デフォルトで動作するようになっているが、若干便利さが重視されており、プロダクション向けに使うには色々考えることが必要だった。
[expoをオレオレバージョニング](https://zenn.dev/terrierscript/articles/2020-08-27-expo-versioning-rules)しながら色々やりくりしていたが、やはりリリースサイクルを早めたくなるとOTAをちゃんと使っていきたいという気持ちになったので考えた。

なお、ExpoのOTAの基礎的な部分については解説を省略している箇所があるため、詳細な点など知りたい場合は下記の記事などが手助けになるだろう

* https://zenn.dev/marin_a___/articles/7eec197a8c3579

## TL;TD

基本的に下記のようなことに気をつければ十分に運用に耐えられそうな気配だった

* build時は`--no-publish`つけることを厳守
  * buildとpublishは明確に分離する
* build・publishのリリースチャンネルは必ず指定する。
  * 両者ともに指定されずに使われるdefaultは利用しない
* publishは`edge`などビルドには利用しないチャンネルに投げる
* buildはpublishとは別なチャンネルを利用する
* `publish:set`によってリリースチャンネルとbuildを紐付ける

## 登場用語

少々語彙がばらつきそうなため、この記事で取り扱う用語を予め下記のようにする

* ビルド
  * アプリストアへアップロード可能なipa,apkのファイル
* リリース
  * Expo上にアップロードされる実際の実行コード
  * publishによってビルド・Expo上へアップロードされ、コードが実行される
  * これは特に正式な名称であることを確認できなかったが、この記事では便宜上「リリース」と呼称する
* publish
  * リリースをアップロードするコマンド、または作業のこと
  * 一度のpublishに対して`publish-id`がexpoから割り振られる
* リリースチャンネル
  * ビルド時に指定する向き先
  * NODE_ENVみたいな概念に近い
  * https://docs.expo.dev/distribution/release-channels/


## なぜbuilid時に`--no-publish`？

expoの`build`コマンドは、アプリをストア配信できるようにするためのものだが、ざっくり分解すると下記の処理が行われる。

1. ビルドバイナリの内部で動作する実行コード（リリース）をexpoのサーバーへアップロード
2. アプリストアへビルド（実際はExpoのビルドキューに追加されるだけ） 

また、`build`については下記の点にも留意したい。

* ビルドはリリースチャンネルを持っており、1対1で紐づく
* ビルドのリリースチャンネルはビルド時に指定され、その後変えることは出来ない

例えば`expo build:ios --release-channel=production`のようにした時、ビルドの前に`publish`チャンネルに最新のリリースが反映される。
まだ`--release-channel=production`に紐付いたビルドがストアで配信されてない場合は特に問題が無いが、すでにアプリを配布している場合、確認前のビルドが`publish`されてリリースされてしまうことになり、これは事故のもとだ。

そこで`--no-publish`が出てくる。`expo build:ios --no-publish`のようにオプションをつけることで、1の「リリースをExpoサーバーへアップロードする」という処理を行わないようにしてくれる。

例えば`expo build:ios --no-publish --release-channel=まだpublishしてないチャンネル`のようなコマンドを打つと下記のようなコマンドが出現してビルドがストップする

```
No releases found. Please create one using `expo publish` first.
```

リリース以前などでは少々このコマンドは面倒に感じるのだが、止めてくれる方が安全な運用ができると考えたほうが良い


### `publish`、`publish:set`、`publish:history`を知る

ここまでの上記で「`publish`」という行為と「`build`」という行為を分離することとなった。
ここからは`publish`についてより深く探っていく。

まず今回の記事で使うのは下記のコマンドとなる。

* `publish` - リリースを生成するコマンド
* `publish:set` - リリースとリリースチャンネルを紐付けするコマンド
* `publish:history` - 直近のリリースの履歴を確認できるコマンド
* `publish:details` - publishIdを指定して1件のリリースを詳細に知れるコマンド
* `publish:roleback` - 直近のリリースを差し戻すコマンド

publishにはrelease-channelを紐付けることができる。`default`でも`development`でも`main`でも何でも良いのだが、ここでは`edge`という名前のチャンネルを最新リリース用としたい。

最新ビルドは下記のコマンドで実行する。

```
$ expo publish --release-channel=edge
```

そして`publish:history`でビルドを確認することができる

```
$ expo publish:history
```

上記コマンドでは人間が少し読みやすいようにテーブル表記になるが、`--raw`をつけてJSON表記にしてみよう（ついでに見やすさのためにjqを通している）。

```
$ expo publish:history --raw | jq -C 
```

```json
  "queryResult": [
    {
      ...
      "channel": "edge",
      "publicationId": "fa6ac07e-229e-4375-8634-b460ee2cd36c",
      ...
      "appVersion": "1.0.2",
      "publishedTime": "2021-09-23T03:47:28.221Z",
      "platform": "ios"
    },
    {
      ...
      "channel": "edge",
      "publicationId": "676174fe-e4bf-490a-bf29-b70ddc685019",
      ...
      "appVersion": "1.0.2",
      "publishedTime": "2021-09-23T03:47:28.221Z",
      "platform": "android"
    },
```

様々情報が出てくるが、ここで重要なのは`publicationId`になる。
ご覧の通り、一度`expo publish`を実行すると`ios`、`android`などapp.jsonで指定したプラットフォームのリリースが行われていることがわかる。

あとはこの`publicationId`を本番で利用する`release-channel`に紐付けることでリリースが完了する。サーバーに近い言い方ならデプロイにあたる作業だ

```
# iosのデプロイ
$ expo publish:set --release-channel=production fa6ac07e-229e-4375-8634-b460ee2cd36c

# androidのデプロイ
$ expo publish:set --release-channel=production 676174fe-e4bf-490a-bf29-b70ddc685019
```

ここでもう一度`publish:history`コマンドを叩いてみよう
```
$ expo publish:history --raw -p ios | jq -C
```

すると下記のように`edge`と`production`にそれぞれ`fa6ac07e-229e-4375-8634-b460ee2cd36c`がリリースされていることが確認できるはずだ

```json
{
  "queryResult": [
    {
      ...
      "channel": "production",
      "publicationId": "fa6ac07e-229e-4375-8634-b460ee2cd36c",
      ...
      "publishedTime": "2021-09-23T03:47:28.221Z",
      "platform": "ios"
    },
    {
      ...
      "channel": "edge",
      "publicationId": "fa6ac07e-229e-4375-8634-b460ee2cd36c",
      ...
      "publishedTime": "2021-09-15T23:28:44.226Z",
      "platform": "ios"
    },
```


### publicationIdをワンラインで取得する

ここまででリリースの仕組みを明らかにしてきた。しかし毎回publicationIdを取得するのは少々面倒なのでコマンドを作ろう。
`publish:history`コマンドには件数を絞り込む`--count`とプラットフォームを絞り込む`--platform`のオプションがあるので、これらを組み合わせるとこのようなコマンドで最新一件のリリースを取得できる

```
$ expo publish:history --raw --count 1 --release-channel=edge --platform=ios 
```

結果にjsonが返却されるので、ここから取得することができるだろう。
`jq`を利用するなら下記のような感じだ

```
$ expo publish:history --raw --count 1 --release-channel=edge --platform=ios \
    | jq -r ".queryResult[0].publicationId"
```

しかしjqが入ってない環境・nodeで完結させたい場合もある。様々コマンドはあるが、今回は`jq.node`を利用してみる

```
$ yarn add jq.node
$ expo publish:history --raw --count 1 --release-channel=edge --platform=ios \
    | yarn -s jqn 'get("queryResult.0.publicationId")'
```

これで一行だけ`publicationId`が出力されるコマンドになったはずだ。


### リリースコマンドを作る

上記のpublicationIdの出力ができたので、リリースコマンドと組み合わせることができるだろう。

```
$ expo publish:set --release-channel=production \
    --publish-id=$(expo publish:history --raw --count 1 --release-channel=edge --platform=ios \
    | yarn -s jqn 'get("queryResult.0.publicationId")')
```
少々複雑すぎるので`npm script`あたりにでも仕込むのが良いだろう

```json
"echo:publish:ios": "expo publish:history --raw --count 1 --release-channel=edge --platform=ios | jqn 'get(\"queryResult.0.publicationId\")'",
"echo:publish:android": "expo publish:history --raw --count 1 --release-channel=edge --platform=android | jqn 'get(\"queryResult.0.publicationId\")'",
"release:production:ios": "expo publish:set --release-channel=production --publish-id=$(yarn -s echo:publish:ios)",
"release:production:android": "expo publish:set --release-channel=production --publish-id=$(yarn -s echo:publish:android)",
```

あとは下記のように実行できるはずだ。

```
$ yarn release:production:ios
$ yarn release:production:android
```


## 実運用でのコマンド

ここまででの処理をまとめてみる

1. (リリースの作成作業）`expo publish --release-channel=edge`でリリースをedgeチャンネルに配布
   * edgeチャンネルを見ている本番ビルドは存在しないので特に問題無い
2. (デプロイの作業)`yarn release:production:ios`で最新ビルドをデプロイ（リリースチャンネルへの紐付け）
   * このタイミングでOTAが実行され、反映される
3. 必要に応じて`expo build:ios`でバイナリを生成・Appストアへの配布
   * 事前にビルド・デプロイを実行することで、初期にバンドルされるリリースが最新版になる

これでリリースとデプロイが分離された運用が可能になりそうだ。

## 欄外

### どのビルドが入ってるか確認するには？

これらOTAを実行してると、少々どれが入っているか分かりづらい。
基本的には
[`expo-updates`](https://docs.expo.dev/technical-specs/expo-updates-0/)の`Updates.manifest`
が利用できる。
残念ながらこのmanifestには`publicationId`が含まれていないのだが、`revisionId`で確認できるので、これらをアプリのどこかで確認できるようにしておくと良い。

なお、各リリースのrevisionIdは`expo publish:details`のコマンドで確認が可能だ。

また、ビルド時にgitリビジョンを`manifest.extra`に含める手もあるだろう。

```js
// app.config.js
export default ({ config }) => {
  return {
    ...config,
    extra: {
      "gitRevision": process.env.GIT_REVISION
    }
  }
}
```

あとは環境変数から渡す。CIなどで実行するようにしても良いだろう。

```
$ GIT_REVISION=$(git log -n 1 --format=%H) expo publish --release-channel=edge"
```

### カナリアリリースをどうするか？

ここで一つ問題が残る。expoのbuildは特定のrelease-channelしか持てないため、開発者のみがアーリーアクセスするような機構を作るのがどうしても難しい側面がある。

思いつく手法としては下記があるだろう。

* A. `release-channel=staging`のビルドを別途生成し、別途stagingビルドを利用する
  * 個人的には以前やっていたが、割とビルド〜デプロイが面倒でやめてしまった
  * 参考: https://zenn.dev/terrierscript/articles/2021-02-28-expo-app-json-flavor
* B. 別なrelease-channelを用意し、次回リリースとしてテストする
  * 実機テストはtestflight等を利用
* C. サーバー側でトグルフラグを用意し切り替え
  * 一部やってみたことはあるが、クライアント側の都度実装の負荷が大きいかも？
* D. `expo-updates`を利用し、サーバーと連携し、Updateする対象を特定のルールで決定
  * ちょっと実装負荷が高くて試せてないが、そのうちやってみたい

結局SDKリリースなどのたびに定期的にストアへのリリースは必要なので、個人的にはBと前述の[オレオレバージョニング](https://zenn.dev/terrierscript/articles/2020-08-27-expo-versioning-rules)と組み合わせが妥当で硬いやり方だなとは感じつつ、いずれにせよこの点に関しては何らかの工夫が必要そうだと感じた
