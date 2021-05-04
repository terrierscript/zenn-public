---
title: cargo-watchでターゲット(--target)を指定したい
emoji: 👀
type: tech
topics:
  - rust
  - cargo
published: true
terrier_date: '2019-09-21T06:06:48.175Z'
terrier_export_from: 'https://blog.terrier.dev/blog/2019/20190921150648-cargo-watch-target'
---

> 記事作成日: 2019/09/21

`-x`オプションでコマンドを監視が出来る便利な[cargo-watch](https://github.com/passcod/cargo-watch/)

```
$ cargo watch -x check
```

ただここで`--target`のようなオプションを渡すとエラーになる

```
$ cargo check --target arm-unknown-linux-musleabihf
```

```
error: Found argument '--target' which wasn't expected, or isn't valid in this context

USAGE:
    cargo watch [FLAGS] [OPTIONS]

For more information try --help
```

# 解決策
今の所issueがオープンしているが、`-x`ではなく`-s`で指定すればとりあえず回避出来る

https://github.com/passcod/cargo-watch/issues/97

```
$ cargo watch -s "cargo check --target arm-unknown-linux-musleabihf"
```
