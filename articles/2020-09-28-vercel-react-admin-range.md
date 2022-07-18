---
title: Vercelとreact-adminを組み合わせるとrangeヘッダでエラーが出るので注意
emoji: 🥀
type: idea
topics:
  - nextjs
  - vercel
  - react
  - javascript
  - reactadmin
published: true
---

React Adminとnext.jsを組み合わせようとしてハマったのでメモ。
~~なお、現在このエラーについては問い合わせ中なので、何らか解決される可能性もある~~ 記事下部に追記しました

# 前提: React Admin

React Adminにおいての標準的なdataProviderの`ra-data-simple-rest`を利用する場合、`Content-Range`ヘッダをページネーションに利用している。そしてChromeへの対応として`Range`ヘッダを飛ばしている。

https://github.com/marmelab/react-admin/blob/7aa5559ef8552605b02dadf277ae2a2f38e86c79/packages/ra-data-simple-rest/src/index.ts#L57-L61


### そもそもRangeヘッダとは

* https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Range
* https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Content-Range

> Range は HTTP のリクエストヘッダーで、サーバーが返すべき文書の部分を示します。

とのことで、「いやページネーションにRangeヘッダ使うのどうなのよ」という気持ちが勝ちつつ、好意的に解釈すれば「ページネーションもサーバーの返すべき文章の部分と言えなくもない」、とギリギリ思うことは出来そう

## エラーの起きる条件

こちらに今回使うサンプルを配置した。といってもほとんど基本テンプレをts化した程度のものになる
* source: https://github.com/terrierscript/example-range-error
* demo: https://example-range-error.vercel.app/api/hello



### ローカルで動かしてみる
これをまずローカルで動かしてみる。これは問題なく動く

```bash
$ curl http://localhost:3000/api/hello
{"name":"John Doe"}
```

次にRangeヘッダをつける。これも当然問題ない

```bash
$ curl http://localhost:3000/api/hello -H "Range: foo=0/1"
{"name":"John Doe"}
```

## Vercel上で動かす

まずは普通に動かす。動く

```bash
$ curl https://example-range-error.vercel.app/api/hello
{"name":"John Doe"}
```

次に問題のRangeヘッダ

```bash
$ curl https://example-range-error.vercel.app/api/hello -H "Range: foo=0/1"
A server error has occurred

INTERNAL_SERVER_ERROR
```

コケた。ちなみにFunctionsのログにも何もログが出ず、その手前でコケている可能性が高い
おそらくだがVercel側が処理しようとしてコケているものと思われる。

# 取り急ぎの対策どうする？

`ra-data-json-server`の方であればRangeヘッダを利用してないので、ひとまずこちらを使えば回避できる

* https://github.com/marmelab/react-admin/blob/cfb937962d4b49753af965b7ee3d2d7035a1e3e3/packages/ra-data-json-server/src/index.ts

# 追記: Vercel社の回答

下記のような回答を頂いた

> I can see that your usage of the Range​header is actually incorrect and was able to get a successful response by adjusting the syntax of the header as documented by MDN.
> Notice that I get only 10 bytes back in my response when I issue the following request for the byte range of 0 - 10 and don't run into any errors:

「それはRangeヘッダの使い方が正しくないから下記みたいにすれば動くよ。とのこと。
で、ですよね。

```bash
$ curl https://example-range-error.vercel.app/api/hello -H "Range: bytes=0-10"
{name: "John Doe"}
```

Rangeヘッダのunitが正しくない場合は無視してくれると期待してたがそうでない実装はありえそうだ。
幸いReact AdminのDataProviderは選べるので、やはり「Rangeヘッダを利用するData Providerを避ける」というのが無難であろう。