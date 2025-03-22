---
title: VSCodeのcopilotに複数のinstructionsファイルを読み込ませる
emoji: 🦛
type: tech
topics:
  - vscode
  - copilot
published: true
---

copilotはデフォルトで`.github/copilot-instructions.md`のファイルは参照してくれる。

ただ、このファイルの中で`foo/bar.md`のように更に別のファイルを参照するような記載をしてもそれらのファイルまでは読み込んではくれないらしく、都合の悪いことが多い。

# VS Codeで設定

[VSCode設定ドキュメント](https://code.visualstudio.com/docs/copilot/copilot-customization)には、下記のような設定を追加できるようになっているのを見つけた。

```
"github.copilot.chat.codeGeneration.instructions": [
  {
    "file": ".rules/rules.md"
  }
],
```

他にも指示出しには以下のような種類がある（が、特に試してない）

* `github.copilot.chat.testGeneration.instructions` - テストコードの生成に対する指示
* `github.copilot.chat.reviewSelection.instructions` - コードレビューに対する指示
* `github.copilot.chat.commitMessageGeneration.instructions` - コミットメッセージの生成に対する指示

またファイルではなくtextで指示を追加するのも出来るっぽい（これも試してない）

# シンボリックリンクを活用する

指示する内容をファイルにまとめる場合、共通のルールファイルなどをプロジェクト間で共有したい場合もあるだろう。
そういった場合はシンボリックリンクを利用すると便利。

```
$ ln -s ~/.config/rules/rules ./.rules
```

プロジェクトに関わらせたくない部分であれば`.gitignore`に追加しておくとかでも良い（`.config/.git/ignore`の方が良いかも？

```.gitignore

.rules/**/*

```

これで共有したいプロジェクト間で同じ設定を使い回すことができる。

# 確認方法

Copiotを動かしたときに `2 参照 使用済み`のような小さな折りたたみ表示がある。これを展開すると読み取っているかどうか確認できる。

「このファイルを読み取ったらXXXと言ってください」みたいにして返事をさせるのも良い