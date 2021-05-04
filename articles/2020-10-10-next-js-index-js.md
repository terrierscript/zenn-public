---
title: next.jsのルーティングはindex.jsを使うと綺麗にまとまる
emoji: 🚆
type: tech
topics:
  - nextjs
  - javascript
published: true
---

next.jsの動的ルーティングを利用すると、下記のようになってしまうことがよくあった

```
.
├── users
│   └── [id].js // 詳細ページの用途
└── users.js    // listページの用途
```

一つぐらいなら気にならないが、要素が増えてくると下記のようになってきてなかなか管理しづらくなってくる

```
.
├── items
│   └── [id].js
├── tags
│   └── [name].js
├── users
│   └── [id].js
├── items.js
├── tags.js
└── users.js
```

と、実は`index.js`が使えるということに気付いた

* https://nextjs.org/docs/api-routes/dynamic-api-routes#index-routes-and-dynamic-api-routes
* https://nextjs.org/docs/routing/introduction#index-routes

```
.
├── items
│   ├── [id].js
│   └── index.js
├── tags
│   ├── [name].js
│   └── index.js
└── users
    ├── [id].js
    └── index.js
```

これでスッキリする