---
title: formatToPartsとObject.fromEntriesで自由に日付や時間をformatする
emoji: 📆
type: tech
topics:
  - javascript
  - typescript
published: true
---

日付をフォーマットしたい場合、モダンブラウザであれば`Intl.DateTimeFormat`が利用できる

* https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat

`locale`やオプションを設定が色々あるので、だいたいは可能だ。

例えばo月x日みたいな表記であればこのようになる

```ts
new Intl.DateTimeFormat("ja-JP",{month:"long", day:"numeric" })
        .format(new Date())
// => 10月20日
```

とはいえどのオプションがどれだったかを覚えてられなかったり、もう少し自由なフォーマットをしたいときもある。

## formatToPartsから取り出す

そこで`formatToParts`を使うとそれぞれの部位の配列として受け取ることが出来る

* https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat/formatToParts

```ts
new Intl.DateTimeFormat("ja-JP")
  .formatToParts(new Date())
```
```json
[
  {"type": "year","value": "2022"},
  {"type": "literal", "value": "/"},
  {"type": "month", "value": "10"},
  {"type": "literal", "value": "/"},
  {"type": "day", "value": "20"}
]
```

ただこのままだとformatするには少し扱いづらい

## `Object.fromEntries`で扱いやすくする

ここから`Object.fromEntires`と組み合わせてやるとObjectとして扱えるので取り回しやすい

* https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Object/fromEntries

```ts

const getFormatParts = (date: Date) => {
  return Object.fromEntries(
      new Intl.DateTimeFormat("ja-JP", {dateStyle:"full", timeStyle:"full"})
          .formatToParts(date)
          .map(({type, value}) => [type,value])
      )
}
```

```json
{
    "year": "2022",
    "literal": "秒 ",
    "month": "10",
    "day": "20",
    "weekday": "木曜日",
    "hour": "20",
    "minute": "58",
    "second": "24",
    "timeZoneName": "日本標準時"
}
```

あとは下記のように利用できる

```ts
const dateParts = getFormatParts(new Date())
const formatted = `${dateParts.month}月${dateParts.day}日 [${dateParts.weekday}]`

// => '10月20日(木曜日)'
```
