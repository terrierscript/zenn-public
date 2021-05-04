---
title: material-uiのDataGridが無限に伸びる時の対処
emoji: 🥙
type: tech
topics:
  - react
  - javascript
  - materialui
published: true
---

[Material UIのData Grid](https://material-ui.com/components/data-grid/)を試してみたところ下記のようにコンポーネントを配置したさい、無限に伸びる現象に遭遇した。

```jsx
<div>
  <div>some title</div>
  <DataGrid
    pageSize={5}
    columns={columns}
    rows={rows} />
</div>
```

## 解決策

もしかしてデータが再更新されてるのか？などを考えuseMemoを使ってみたりするもダメ。

そこでExampleとにらめっこしてみると、どうやら親を`<div style={{ height: 400, width: "100%" }}>`と囲っているらしいことがわかった。

```jsx
<div style={{ height: 400, width: "100%" }}>
  <DataGrid
    pageSize={5}
    columns={columns}
    rows={rows} />
</div>
```

確かにこれでInfinity loopは収まった。しかしいちいちheightを指定しないといけないとはどうにかならんか？と[API](https://material-ui.com/api/data-grid/)の方を見てみると`authHeight`という項目があるのがわかった

```jsx
<DataGrid
  autoHeight
  pageSize={5}
  columns={columns}
  rows={rows} />
```

これでも無限延長が収まることがわかった。

ただし[Auto Height](https://material-ui.com/components/data-grid/rendering/#auto-height)の項目を見ると非表示領域を隠すVirtualizationしないので大規模なデータには向いてない、とのこと。

