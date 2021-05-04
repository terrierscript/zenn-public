---
title: cheerioをやめてpuppeteerでローカルファイルもスクレイピングを完結させる
emoji: ✂️
type: tech
topics:
  - cheerio
  - puppetter
  - javascript
published: true
---

ちょっとした分析でスクレイピングをする際、だいたい普段こんな具合で行っている

1. 必要なページのリストをなんやかんやでリスト化する
2. リストをwgetやcurlで取得する(`$ wget -i list.txt`が便利)
3. 手元に持ってきたファイルを`iconv-lite`と`cheerio`で処理

クローリングしながらスクレイピングをする人も多いと思うが、個人的には処理ミス時の再トライや、新たに別な値を再取得したくなるようなことも少なくないのでクローリングと実際のスクレイピングは分離するようにするのはオススメだ。

例えばHTMLファイルのテーブルから値を抽出したいならこんな具合

```js
const cheerio = require("cheerio")
const iconv = require("iconv-lite")

const file = fs.readFileSync("./file.html")
const ff = iconv.decode(file, "ShiftJIS").toString()
const $ = cheerio.load(ff)
let trs = []
$("table tr").each(function (_, tr) {
let tds = []
  $(tr).children("td").each(function (_, td) {
    tds.push($(td).text().trim())
  })
  trs.push(tds)
})

```

cheerioはjQueryライクなのが魅力だったものの、もはやjQueryを触らなくなって久しく、そろそろ忘れてしまっているし、`map`のような処理をしたいときに上記のように`each`で処理しなければならなかったり、脱出したい気持ちが強まってきていた

### puppeteerに全部任せる

puppeteerは当然ちょこちょこ使っていたものの、これまでどちらかといえばe2eの用途などリッチなものが多かったり、ローカルファイルの処理には適してない印象が多かった。

しかし調べるとどうやら`setContent`を利用すればローカルファイルも処理出来そうだという情報を発見した

https://stackoverflow.com/questions/47587352/opening-local-html-file-using-puppeteer/52789802#52789802

これを利用するとこんな具合になる

```js

const file = fs.readFileSync("./file.html")
// 本当はiconvもpuppeteerに任せたかったが、ちょっと難しそうで断念
const encoded = iconv.decode(file, "ShiftJIS").toString()

// ブラウザをセットアップ
const browser = await puppeteer.launch()
const page = await browser.newPage()
// ページを設定
await page.setContent(encoded.toString())

// スクレイピング
const table = await page.$("table")
const trs = await table.$$("tr")
const data = await Promise.all( trs.map(async tr => {
  return await tr.$$eval("td", (nodes) => {
    // このクロージャ内部はchrome側で実行される
    return nodes.map(node => node.innerText)
  })
}))
browser.close()

```

ちょっとハマりどころとしては`$$eval`の内部はchrome側で実行される部分になる（その他、`$eval`,`evaluate`は同様とのことらしい
そのためこの内部は
当然`console.log()`など行ってもnode側には表示されない。

もしデバッグしたければ下記のようにすると確認しやすい。

```js
const browser = await puppeteer.launch({
  headless: false,
  devtool: true
})
```


