---
title: javascriptで全角は2文字、半角は1文字な文字長(幅)の計算をワンライナーで行う
emoji: 🔢
type: tech
topics:
  - javascript
published: true
---

まずはワンライナーで

```js
[..."abcあ😀"].reduce( (count,char) => count + Math.min(new Blob([char]).size,2) , 0)

> 7
```

ちょっと読みづらくなってしまうので、説明のために関数化してみる

```js
const strWidthLen = (str) => {
  return [...str]
    .reduce( (count,char) => {
      const len = Math.min(new Blob([char]).size,2)
      return count + len
    }, 0)
}

```

* `[...str]`は`string.split("")`と同義。文字列を配列にしている
* reduceは合計計算のため
* `new Blob([文字列])`にてマルチバイトが計算出来る。
  * ただし絵文字などは4などが計算される。
  * そのため、これを`Math.min`を利用することで1 or 2と丸める
* アラビア語などはちょっと未検証（そもそも2文字としてカウントするべきなのかがわからない…）
* 半角カナ等は考慮していない ([comment](https://zenn.dev/terrierscript/articles/2020-09-19-multibyte-count#comment-d7b59b2ddf0025bbacd3))
参考： https://stackoverflow.com/questions/5515869/string-length-in-bytes-in-javascript

