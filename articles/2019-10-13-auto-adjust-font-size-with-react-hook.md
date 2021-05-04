---
title: 内部要素のサイズに合わせてfont-sizeを調整するcomponentをhooksで作る
emoji: 🍄
type: tech
topics:
  - reacthooks
  - react
  - javascript
published: true
terrier_date: '2019-10-13T13:33:28.451Z'
terrier_export_from: https://blog.terrier.dev/blog/2019/20191013223328-auto-adjust-font-size-with-react-hook
---

> 記事作成日: 2019/10/13

作ったものメモ。

@[tweet](https://twitter.com/terrierscript/status/1183039483702202368)

ふとiOSで`adjustsFontSizeToFitWidth`という横幅に合わせてフォントサイズが変わるのがあったな、というのを思い出したので再現してみた。

```js
import React, { useState, useRef, useLayoutEffect, useEffect } from "react"
import stringWidth from "string-width"
import styled from "styled-components"

const Container = styled.div`
  width: 100%;
`

const AutoSizedButton = ({ text }) => {
  const ref = useRef()
  const [width, setWidth] = useState(0)
  const [fontSize, setFontSize] = useState("auto")
  useEffect(() => {
    const sizePx = (width / stringWidth(text)) * 2
    setFontSize(`${sizePx}px`)
  }, [width, text])

  useLayoutEffect(() => {
    // @ts-ignore
    const obs = new ResizeObserver((e) => setWidth(e[0].contentRect.width))
    obs.observe(ref.current)
    return () => obs.disconnect()

  }, [])
  return (
    <Container ref={ref} fontSize={fontSize}>
      {text}
    </Container>
  )
}
```

[`ResizeObserver`](https://developer.mozilla.org/ja/docs/Web/API/ResizeObserver)を利用している。
まだこれも不安定なツールなので、そこまで実用的なものではない。

フォントサイズの計算として、`string-width`を利用している。昔は`encodeURIComponent`での計算方法などがあったが、今は3byteで判定されるので使えなくなっている。

カスタムhooksとして切り出すならこんな感じだろう
（が、refsを渡すのはいまいちなので切り出しに向いてない）

```js
const useAutoFontSize = (targetRef, text) => {
  const [width, setWidth] = useState(0)
  const [fontSize, setFontSize] = useState("auto")
  useEffect(() => {
    const sizePx = (width / stringWidth(text)) * 2
    setFontSize(`${sizePx}px`)
  }, [width, text])
  
  useLayoutEffect(() => {
    // @ts-ignore
    const obs = new ResizeObserver((e) => setWidth(e[0].contentRect.width))
    obs.observe(targetRef.current)
    return () => obs.disconnect()
  }, [])
  return fontSize
}
```

