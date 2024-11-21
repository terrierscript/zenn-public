---
title: next.js 15に備えてPageやLayoutに独自型定義をつけておく
emoji: 🛁
type: tech
topics:
  - nextjs
  - typescript
published: true
---

[next.jsのversion 15](https://nextjs.org/docs/app/building-your-application/upgrading/version-15)より、`page.js`や`layout.js`の`params`が非同期になった。

基本的には`params`が同期的になっている場合、dev実行時に警告を出したり、build時に失敗してくれるのだが、下記のようにすり抜けるケースがいくつか見られた

* `page.app.tsx`のように`pageExtensions`を変更している場合にはdev実行時のみ警告され、ビルド時には警告されない
* `layout.tsx`の場合、ファイルの配置（？）によって警告されないことがある（詳細不明）

今後修正される可能性が高いが、上記のようなキャッチしきれない場合があるようだった。

また、これら警告がされなかった場合で、うっかり同期的な利用が混在してしまっても、v15においては後方互換があるため、即時にエラーされたりすることはない。
とはいえ将来的に廃止されるものなので、可能な限り発見可能な状態にしておきたい。

## TypeScriptの独自型を定義して発見しやすくする
v14からマイグレーションする場合、下記のように14用の定義を先にしておいて、バージョンアップに合わせて型定義も変更すると検知できる

```ts
// next v14用の定義
export type AsyncAppLayout<TParams = {}> = (props: {
  children: React.ReactNode
  params: TParams
}) => Promise<React.ReactNode>

export type SyncAppLayout = (props: {
  children: React.ReactNode
}) => React.ReactNode

export type AsyncAppPage<TParams = {}, TSearchParams = {}> = (props: {
  params: TParams,
  searchParams: TSearchParams
}) => Promise<React.ReactNode>

export type SyncAppPage = () => React.ReactNode
```


```ts
// next v15用の定義。アップデートと同時に書き換える
export type AsyncAppLayout<TParams = {}> = (props: {
  children: React.ReactNode
  params: Promise<TParams>
}) => Promise<React.ReactNode>

export type SyncAppLayout = (props: {
  children: React.ReactNode
}) => React.ReactNode

export type AsyncAppPage<TParams = {}, TSearchParams = {}> = (props: {
  params: Promise<TParams>,
  searchParams: Promise<TSearchParams>
}) => Promise<React.ReactNode>

export type SyncAppPage = () => React.ReactNode
```

使い方はこんな具合

```tsx
const Layout: AppLayout<{ name: string }> = ({
  children,
  params,
}) => {
  const resolvedParams = await params
  return (
    <Box>
      <Box>name: {resolvedParams.name}</Box>
      {children}
    </Box>
  )
}
export default Layout
```


