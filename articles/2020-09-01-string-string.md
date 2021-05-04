---
title: 'string|string[] な値をいい感じに処理する'
emoji: 🦥
type: tech
topics:
  - typescript
  - javascript
published: true
terrier_export_from: https://blog.terrier.dev/blog/2020/20200901232406-string-string/
---

next.jsなどをいじってると、よく`string|string[]`みたいな値を取り扱うことがあるときの処理を忘れがちなのでメモ。

こういうときは`flat()`して`map()`するといい感じに処理出来る（似ているが`flatMap()`ではない）

```ts
function appendC(strOrArr : string | string[]) : string[] {
  return [strOrArr].flat().map( item => item + "c" )
}

appendC("a")  // ["ac"]
appendC(["aa", "bb"])  // ["aac", "bbc"]
```

例えば配列を更に返したいようなケースなら`flatMap`に変えられる

```ts
function splitItems(strOrArr : string | string[]): string[]{
  return [strOrArr].flat().flatMap( item => item.split(",") )
}

splitItems("a,b")  // ["a","b"]
splitItems(["aa,bb", "bb,cc"])  // ["aa","bb", "bb", "cc]
```

ちなみに`[1,[1,2,[1,2,3],2],[3,2]]`のような深いArrayならば`flat(Infinity)`を使うことも出来る

```ts
[deepArray].flat(Infinity).map(...
```

