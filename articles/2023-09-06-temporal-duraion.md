---
title: Temporal.Duraionから◯日前みたいなlocale対応した文字列にする
emoji: ⏱️
type: tech
topics:
  - javascript
  - typescript
published: true
---

# Durationの抽出
まず二つのDateからDurationを抽出する。
今回はInstantに変換してsinceにして差分を取得する

```ts
const startInstant = Temporal.Instant
  .fromEpochMilliseconds(new Date('2021-01-01 00:00:00').getTime())
const targetInstant = Temporal.Instant
  .fromEpochMilliseconds(new Date('2023-01-14 12:34:56').getTime())

const relative = startInstant.toZonedDateTimeISO(Temporal.Now.timeZoneId())
const duration = targetInstant.since(startInstant).round({
  smallestUnit: "second",
  largestUnit: "year",
  relativeTo: relative,
})
```

Durationは、ISO8601の形式になっており、このままだと扱いづらい。

```ts
console.log(duration.toString())
// P2Y13DT12H34M56S
```

# Durationから「◯日前」みたいなのを生成する

Durationから最大の値を取り出して表示したい場合（GitHubの最終更新日みたいなアレ）をやりたい場合は、`RelativeTimeFormat`を利用すると良い。

```ts
const units = ["year", "month", "day", "hour", "minute", "second"] as const

// durationから各unitの値を取り出して、値が存在する最初の単位を取得する
const unitValue = units.map(unit => {
  const multipler = `${unit}s` as const
  const value = duration[multipler]
  return { unit, value }
}).find(item => item.value > 0)

if (!unitValue) {
  return
}

const durationBefore = new Intl.RelativeTimeFormat(locale).format(unitValue.value, unitValue.unit)
```

```ts
console.log(durationBefore)
// => 3 か月前
```
これでlocaleに対応した◯日前　みたいなものができる

# Durationから値を全部フォーマットしたい場合

最大の単位ではなくすべての値を細かく表示したい場合は、`RelativeTimeFormat`だと少し足りない。

`Intl.DurationFormat`があれば一発でそれを利用できそうだが、今のところ利用できる環境もpolyfillも薄いので、自前でフォーマットする必要がある。
localeが不要などであれば愚直に対応表のようなものを用意してしまえばいいのだが、今回は`Intl.NumberFormat`を利用してみる。

```ts
const locale = "ja-JP"

// 値とのRecordを取り出す
const units = ["year", "month", "day", "hour", "minute", "second"] as const
const unitValues = units.map(unit => {
  const multipler = `${unit}s` as const
  const value = duration[multipler]
  return { unit, value }
})

// 取り出した値をフォーマットする
const formattedDuration = unitValues.map(({unit,value}) => {
  if (value < 1) return null
  return new Intl.NumberFormat(locale, {
    style: "unit",
    unit,
  }).format(value)
}).filter(item => item !== null).join(" ")
```

このようにすると下記のような形式が取れる
```
2 年 13 日 12 時間 34 分 56 秒
```

## 単位と文字の空白を消す

`Intl.NumberFormat`は日本語でも数値と単位の間に空白を入れてしまうので、若干気持ち悪い見た目の場合もある。

```
2年 13日 12時間 34分 56秒
```

のようにしたい場合は

```ts
new Intl.NumberFormat(locale, {
  style: "unit",
  unit,
}).format(value).replace(" ","")
```

と`replace`で消してしまうか、または`formatToParts`を利用するならちょっと遠回りだが

```ts
const [unit] = new Intl.NumberFormat(locale, {
  style: "unit",
  unit,
}).formatToParts(value).filter(item => item.type === "unit").map(item => item.value)
return `${value}${unit}` // localeによってはこのルールだと不都合な可能性はある
```
というのも考えられる。

# 補足

## ISO8601 Durationの変換

今回は自前でDurationから値を取り出したが、下記のようなライブラリを利用する手段もある。ただし、localeを考慮してくれるようなものはなさそうだった。
- [tinyduration](https://github.com/MelleB/tinyduration)
- [duration-fns](https://github.com/dlevs/duration-fns)

`duration.toString()`によってISO8601形式が取得できるので、上記ライブラリで再変換する

## unitの取り出し

`units`は無理をすれば`Intl.DateTimeFormat`から再現することも出来なくはない。
ただし、`second`が入ってこなかったり余計なキーが入ってきたりするので、そんなにおすすめはしない。

```tsx
const units = new Intl.DateTimeFormat("ja-JP", {
  dateStyle: "short",
  timeStyle: "short",
})
  .formatToParts(new Date())
  .filter(value => value.type !== "literal").map(value => value.type)
// => ["year", "month", "day", "hour", "minute"]
```