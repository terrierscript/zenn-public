---
title: ちょっとだけコンパクトなJSON.stringifyをする
emoji: 👌
type: tech
topics:
  - javascript
  - json
  - nodejs
published: true
---
JSONをファイルに保存したいときなど、`JSON.stringify`を使うのが一般的だろう。

```js
const obj = {a:{b:{c:{e:{f:["g",{h:["i"]}]}}}}, b:{c:"r"}}
JSON.stringify(obj)
// "{"a":{"b":{"c":{"e":{"f":["g",{"h":["i"]}]}}}},"b":{"c":"r"}}"
```

`JSON.stringify`は`JSON.stringify(obj,null,2)`のように第三引数に数字を入れることでスペースを入れることができる。

ただしこれだとすべてネストされてしまう。

```json
{
  "a": {
    "b": {
      "c": {
        "e": {
          "f": [
            "g",
            {
              "h": [
                "i"
              ]
            }
          ]
        }
      }
    }
  },
  "b": {
    "c": "r"
  }
}
```

大きいJSONをファイルに保存したいときなど、spacerナシだとエディタで開いたときに使いづらいし、spacer=2にすると上記のようにネストが深すぎて扱いづらくなる。

## しょうがないので自作する

とりあえず自分の場合は一段ネストしてくれるぐらいで十分だったので、下記ぐらいで十分そうだった。

```js
const jsonCompactStringify = (obj) => {
  const [s,e] = Array.isArray(obj) ? [`[`, `]`] : [`{`,`}`]
  return [s,
    Object.entries(obj)
      .map(([k, v]) => `  "${k}" : ${JSON.stringify(v)}`)
      .join(",\n")
  , e].join("\n")
}
```

```js
jsonCompactStringify(obj)
// {
//   "a" : {"b":{"c":{"e":{"f":["g",{"h":["i"]}]}}}},
//   "b" : {"c":"r"}
// }
```

object/array以外は未対応。ネストを深くしたい場合は上記を再帰的に回すことになるだろう

---

ちなみに[nodeの`util.inspect`](https://nodejs.org/api/util.html#util_util_inspect_object_showhidden_depth_colors)などには`compact`オプションなどがあるが、こちらはあくまでJavasScriptとしての表現をするものなので、JSONに変換するのはエスケープ等を考えると少々難しそうだった
