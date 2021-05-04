---
title: MacのVSCodeで絵文字をちゃんと表示したい
emoji: 😶
type: idea
topics:
  - vscode
published: true
---

VSCodeで、絵文字を使うと一部の絵文字が置換されているのが気持ち悪かった。

## デフォルトだと？

```
Menlo, Monaco, 'Courier New', monospace
```

![](https://user-images.githubusercontent.com/13282103/115132678-37875f80-a03d-11eb-919e-c5dbc698ee74.png)

## 'Apple Color Emoji'を追加する

[#32840](https://github.com/microsoft/vscode/issues/32840#issuecomment-389704418)に挙げられているのは`Apple Color Emoji`を追加して下記のようにするものだ

```
Menlo, Monaco, 'Courier New', monospace, 'Apple Color Emoji'
```
しかしこれだと一部絵文字は処理されてない

![](https://user-images.githubusercontent.com/13282103/115132677-37875f80-a03d-11eb-90e8-51f0e9670908.png)

## `Menlo`を削る

[#118905](https://github.com/microsoft/vscode/issues/118905#issuecomment-803461280)を見ると、Menloが起因してるようなのでこれを削ってみる

```
Monaco, 'Courier New', monospace, 'Apple Color Emoji'"
```

![](https://user-images.githubusercontent.com/13282103/115132709-6998c180-a03d-11eb-92e6-96d89434e58b.png)

ということで意図通りに出来た。これを設定すると良さそうだ。

![](https://user-images.githubusercontent.com/13282103/115132673-35bd9c00-a03d-11eb-9674-c2e40dbef370.png)

