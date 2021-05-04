---
title: JavaScriptで億とか万とかに変換したいときはNumberFormatのnotation:compactが便利
emoji: 🐡
type: tech
topics:
  - javascript
  - intl
published: true
---

ふとしたきっかけで[NumberFormatにnotation: "compact"](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat/NumberFormat#scientific_engineering_or_compact_notations)というオプションを見つけて便利だった。

```js
const fmt =  new Intl.NumberFormat("ja-JP",{ 
    notation: "compact",
})

fmt.format(BigInt(64 ** 8) )
// => "281兆"
```

とりあえずざっくり数値を出したいときにとても便利。最高そう。

### オプションを組み合わせる
詳しくは上記MDNが詳しいが、その他オプションと組み合わせることで色々調整も可能だ

```js
// 何も設定しないといい感じに小数点計算してくれる
new Intl.NumberFormat("ja-JP",{ notation: "compact"}).format(BigInt(433333333))
// =>  "4.3億"

// 小数点表記をさせたくなければmaximumFractionDigitsを0にする
new Intl.NumberFormat("ja-JP",{ notation: "compact", maximumFractionDigits:0 }).format(BigInt(433333333))
// => "4億"
```

```js
// デフォルトでは4桁区切りでカンマを入れる
new Intl.NumberFormat("ja-JP",{ notation: "compact"}).format(BigInt(433333333333333333))
// => "433,333兆"

// この挙動を抑えるにはuseGrouping:falseにする
new Intl.NumberFormat("ja-JP",{ notation: "compact",useGrouping:false}).format(BigInt(433333333333333333))
// => "433333兆"

```


### formatToParts
更に`formatToParts`を使うとこれらの値をバラバラに受け取れる

```js
JSON.stringify(
  new Intl.NumberFormat("ja-JP",{ notation: "compact"})
    .formatToParts(100000), null, 2)
```
```
"[
  {
    "type": "integer",
    "value": "10"
  },
  {
    "type": "compact",
    "value": "万"
  }
]"
```

```js
JSON.stringify(
  new Intl.NumberFormat("ja-JP",{ notation: "compact"})
    .formatToParts(155500000), null, 2)
```
```
"[
  {
    "type": "integer",
    "value": "1"
  },
  {
    "type": "decimal",
    "value": "."
  },
  {
    "type": "fraction",
    "value": "6"
  },
  {
    "type": "compact",
    "value": "億"
  }
]"
```

## 応用編: ちゃんと最後の桁まで出したかったら？
雑に出したいときはこれで十分だが、もうちょっと細かく出したいときには`formatToParts`でもう一工夫必要だ[^2]

[^2]: 有益かどうかはおいといて・・・

今回は個人的な趣味で再帰を使って書いているが。whileなどでもおそらく書けるだろう。
また、今回のコードはそれほど深く検証してないので、ブラウザ等によって挙動が変わることも想定できる。
ご利用の際はご注意いただきたい

```js
function fullFormat(number) {
  const formatter = new Intl.NumberFormat("ja-JP",{ 
    notation: "compact",
    useGrouping: false,
    maximumFractionDigits: 0
  })
  const fmt = (number, result = []) => {
    const bigIntNum = BigInt(number)
    const [num,notation] = formatter.formatToParts(bigIntNum)
    const numStr = bigIntNum.toString()
    if(notation === undefined){
      return [...result, numStr].join("")
    }
    const dig = num.value.length
    const value = numStr.slice(0,dig)
    const next = numStr.slice(dig)
    
    return fmt(next, [...result,`${value}${notation.value}`])
  }
  return fmt(number)
}
```

前述したように、`maximumFractionDigits: 0`と`useGrouping:false`をすることで数値のみが取れるようにしている。
ただ、`maximumFractionDigits:0`にしてしまうことで数値自体は四捨五入が発生してしまい使えなくなってしまうので、桁数のみを利用し`split`で切り取ることにした [^1]

それと指数表記になってしまうと一瞬で死ぬのでBigIntに変換するようにしている。このへんはもうちょっと上手いやり方があるかもしれない

[^1]: `maximumFractionDigits`を引き伸ばしてformatToPartsの結果を利用するとかも考えたが、そうするとformatterを再初期化する必要などがありそうだったのでやめた


こんな感じのテストコードで試してみる

```js
Array.from({ length: 15 }).map((_, i) => {
  const num = BigInt(64 ** i)
  console.log(`64 ** ${i} : ${64 ** i} | ${num} => ${fullFormat(BigInt(64 ** i))}`)
})
```

```
64 ** 0 : 1 | 1 => 1
64 ** 1 : 64 | 64 => 64
64 ** 2 : 4096 | 4096 => 4096
64 ** 3 : 262144 | 262144 => 26万2144
64 ** 4 : 16777216 | 16777216 => 1677万7216
64 ** 5 : 1073741824 | 1073741824 => 10億7374万1824
64 ** 6 : 68719476736 | 68719476736 => 687億1947万6736
64 ** 7 : 4398046511104 | 4398046511104 => 4兆3980億4651万1104
64 ** 8 : 281474976710656 | 281474976710656 => 281兆4749億7671万656
64 ** 9 : 18014398509481984 | 18014398509481984 => 18014兆3985億948万1984
64 ** 10 : 1152921504606847000 | 1152921504606846976 => 1152921兆5046億684万6976
64 ** 11 : 73786976294838210000 | 73786976294838206464 => 73786976兆2948億3820万6464
64 ** 12 : 4.722366482869645e+21 | 4722366482869645213696 => 4722366482兆8696億4521万3696
64 ** 13 : 3.022314549036573e+23 | 302231454903657293676544 => 302231454903兆6572億9367万6544
64 ** 14 : 1.9342813113834067e+25 | 19342813113834066795298816 => 19342813113834兆667億9529万8816
```
目検した感じそこそこいい感じに出てそうだ[^3]


[^3]: 本当は京とか恒河沙とか出てほしい気持ちもあるけどそこまではやってくれるのはなさそう