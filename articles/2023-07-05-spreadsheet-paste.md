---
title: Spreadsheetのデータを貼り付けてパースできるtextareaを作る
emoji: 📋
type: tech
topics: [TypeScript, JavaScript, Spreadsheet, React]
published: true
---

Spreadsheetからのデータを受け取りたい場合、通常はCSVなどに変換すれば良いのだが、改行や画像を取り込む場合を考えると、spreadsheetから取り込めるようなものが欲しいケースがあった。
色々調べるとできそうだったので、その方法をまとめる。

## 完成物
言葉だけだと分かりづらいので完成物から。

今回のサンプルとしては例えば下記のようなデータを
![](https://storage.googleapis.com/zenn-user-upload/639d9cec9de7-20230705.png)

下記のようにパースして取得できるものをゴールとする
![](https://storage.googleapis.com/zenn-user-upload/2405e5882fac-20230705.png)


* [codesandbox](https://codesandbox.io/s/headless-shape-snj65s)

## 作り方
### textareaで取得する

`textarea`の`onPaste`イベントで`clipboardData`から取得できる。
ここでさらに`getData("text/html")`とするとHTMLの形式で取得できる。

```tsx
<textarea readOnly onPaste={(e) => {
  const pastedHtml = e.nativeEvent.clipboardData?.getData("text/html")
  if (!pastedHtml) {
    return
  }
  parse(pastedHtml)
}} />
// <meta charset='utf-8'><google-sheets-html-origin><style type="text/css"><!--td {border: 1px solid #cccccc;}br {mso-data-placement:same-cell;}--></style><table xmlns="http://www.w3.org/1999/xhtml" cellspacing="0" cellpadding="0" dir="ltr" border="1" style="table-layout:fi...
```
textareaはただの受け口のUIなので`readOnly`としている。
画面全体で受け口にしたければ`addEventListener('paste',...)`などを使っても良いだろう

### 受け取ったHTMLをパースする
ここからHTMLのパースに入る。やり方は色々あるが、今回はほぼ自分用の機能だったので、[DOMParser](https://developer.mozilla.org/ja/docs/Web/API/DOMParser)の機能をそのまま利用する形を取った。
もし一般ユーザーに使ってもらうなど、セキュリティを気にしたいケースであれば、ライブラリを利用したり、サーバーサイドで行うなどをすると良いだろう

* [html5](https://www.npmjs.com/package/html5)や
* [cheerio](https://www.npmjs.com/package/cheerio)
* [parse5](https://www.npmjs.com/package/parse5)

まずDOMから`tr > td`の要素を探し、処理していく。
今回はSpreadsheetからのHTML限定で作成しているので、`table`が複数ある場合などは考慮していない

```ts
const parseRichText = (html: string) => {
  const dom = new DOMParser().parseFromString(html, "text/html")
  return Array.from(dom.querySelectorAll("tr")).map(trItems => {
    return Array.from(trItems.querySelectorAll("td")).map(tdItems => {
      // 後述
      return traverseCell(tdItems)
    })
  })
}
```

セルの要素は下記のように処理した。画像やリンクあたりを処理できるようにしている。
ここはその後にしたい処理によっても変わってくる部分なので、一つの参考として見てもらえればと思う。

```ts
type ParsedNode = { type: string, value: string }
type ParsedCells = ParsedNode[]

const traverseCell = (item: ChildNode): ParsedCells => {
  const children = item.childNodes
  if (children.length < 1) {
    // 通常のtext処理
    if (item.nodeType === item.TEXT_NODE && item.textContent) {
      return [{ value: item.textContent, type: "text" }]
    }
    // 改行の取り扱い
    if (item.nodeType === item.ELEMENT_NODE && item.nodeName === "BR") {
      return [{ value: "\n", type: "text" }]
    }
    // リンクの取り扱い
    if (item instanceof HTMLAnchorElement) {
      const href = item.getAttribute("href")
      if (href) {
        return [{ value: href, type: "link", }]
      }
    }
    // 画像の取り扱い
    if (item instanceof HTMLImageElement) {
      const src = item.getAttribute("src")
      if (src) {
        return [{ value: src, type: "image" }]
      }
    }
    return []
  }
  return Array.from(children.values())
    .map((child: ChildNode) => traverseCell(child))
    // セル内部のHTML構造自体は今回不要だったので、flatすることで除去している
    .flat(10)
}
```

これで完成物のようなSpreadsheetからのHTMLをパースして、データを取得できるようになった。
