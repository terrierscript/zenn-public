---
title: "標準機能でDateをformatする色々（toLocaleString/Intl.DateTimeFormat/etc)"
emoji: "⏰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript"]
published: true
---


[moment.jsがメンテナンスモード](https://momentjs.com/docs/#/-project-status/)になって、「とはいえdayjsとか入れるのも億劫だしチャチャっとvanillaになんとかできないかな」と思って調べた。


# いちばんスマートなやり方

```js
(new Date()).toLocaleString("ja-JP", { timeZone: "Asia/Tokyo" })
```
```
> "2020/9/17 21:00:00"
```

昔は自前する必要があったがいつのまにかそんな必要はなくなっていた。良い世の中…

* https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Date/toLocaleString

# その他の方法
結構色々他の手法もあって惑わされたので、他のちょっと遠回りな手法も下記に記載

## toISOString
```js
(new Date()).toISOString()
```
```
> "2020-09-17T12:00:00:000Z"
```
簡易だけどタイムゾーン指定できない

* https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Date/toISOString

## Intl.DateTimeFormat
```js
new Intl.DateTimeFormat('ja-JP', {year: "numeric", month: "2-digit", day: "2-digit", hour: "2-digit", minute: "2-digit", second: "2-digit", timeZone: "Asia/Tokyo"}).format(new Date())
```
```
> "2020/09/17 21:00:37"
```

一番ナウいやり方。かなり細かく指定出来るので需要はあるが、デフォルトでは日付しか表示しないので「とりあえず日付のわかりやすくしたい」程度だとちょっと長い

* https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat

## date.toLocaleDateString & date.toLocaleTimeString

```js
const date = new Date()
const dateString = `${date.toLocaleDateString("ja-JP", { timeZone: "Asia/Tokyo" })} ${date.toLocaleTimeString("ja-JP", { timeZone: "Asia/Tokyo" })}`
```

```
> "2020/9/17 21:03:08"
```

フルでDateTimeにしたいときは冗長なだけだが、もし日付か時間をバラで出したい場合は使える

* https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Date/toLocaleDateString
* https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Date/toLocaleTimeString

### その他

* [toLocaleStringでISOに近いフォーマットにしたいときはsv-SEにする](https://zenn.dev/terrierscript/articles/2020-09-19-time-sv-se)