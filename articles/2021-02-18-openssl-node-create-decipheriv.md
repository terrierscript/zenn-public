---
title: opensslで暗号化したファイルをnode.jsのcreateDecipherivで復号する
emoji: 🔑
type: tech
topics:
  - nodejs
  - openssl
published: true
---

[VercelにGCPのキーを渡すのをどうにかしたい](https://zenn.dev/terrierscript/scraps/241451e4595c1d)方法について検討していたところ[自前でencrypt](https://leerob.io/blog/vercel-env-variables-size-limit)する例を見つけた。

この例だとオンラインの変換ツールを使っているのだが、もう少しスマートに`openssl`コマンドで暗号化ファイルを生成すればもうちょっといい感じに出来るのではないかと色々試してみた

# 手順サンプル

## 下準備
まず秘匿化したいファイルを生成

```
$ echo "this is secret" > rawfile.txt
```

これをopensslでencodingする。

```
$ openssl enc -aes-256-cbc -a -in rawfile.txt -out encrypted.txt -k passphrase -p
```

* `enc`でエンコードに指定
* `-aes-256-cbc` でアルゴリズム指定
  * keyを短くしたい場合は`-aes-128-cbc`でも良いだろう
* `-a`でbase64平文で出力。`-A`にすればbase64 binaryになる
* `-k passphrase`でパスフレーズを入力。今回はわかりやすさのためにCLIで指定してるが、本来しないほうが良い。
* `-p`で下記のように`key`/`iv`が出力される

```
salt=4FB66F28D1694A76
key=BB2C6AE86D97EB01D4CC0C9A54EBC024A195B54158C7D07AA3FDA24430F25286
iv =AA3789F28406951F8C675F05AF672B45
```

* passphraseとsaltによってkeyとivが生成されているので、これを`-p`で出力せずにnode側で計算することも出来そうだったが、今回は諦めた。
* saltを固定したい場合は`-S 4FB66F28D1694A76`のようにすると良い。


## コード
これを復号化するコードを書いてみたい。

```js
// decrypt.js

const crypto = require("crypto")
const fs = require('fs')

const algorithm = "aes-256-cbc" 
const key = process.env.DECRYPT_KEY
const iv = process.env.DECRYPT_IV

const decrypt = (key, iv, source) => {
  const decipher = crypto.createDecipheriv(
    algorithm,
    Buffer.from(key, "hex"),
    Buffer.from(iv, "hex")
  )
  // ↓ここポイント。`slice`でsalt部分を消す。
  const data = Buffer.from(source, "base64").slice(16)
  const start = decipher.update(data)
  const final = decipher.final();
  const result = Buffer.concat([start, final]).toString("utf8")
  return result
}

// Base64のファイルだが、データ自体はテキストファイルなのでutf8で開く
const source = fs.readFileSync("encrypted.txt", { encoding: "utf8" })
const result = decrypt(key,iv, source)

console.log("decrypt:", result)
```

一番厄介なのが`.slice(16)`している部分。opensslの仕様として先頭にSaltが埋め込まれてるっぽい（調べたがいまいちわからなかった）
`openssl`で`-nosalt`のオプションを付ければこの処理が不要にはなるが、後方互換用のオプションなのでおそらくやめたほうが良さそうだ。

そしてこれを環境変数で指定して実行してみる

```
$ DECRYPT_KEY=BB2C6AE86D97EB01D4CC0C9A54EBC024A195B54158C7D07AA3FDA24430F25286 \
DECRYPT_IV=AA3789F28406951F8C675F05AF672B45 \
node decrypt.js
```

下記のように復号化が確認できるだろう。

```
decrypt: this is secret
```

vercelで利用したい場合は`DECRYPT_KEY`と`DECRYPT_IV`を埋め込めが復号化出来るはずだ