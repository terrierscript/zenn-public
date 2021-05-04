---
title: React Hooksでモバイルデバイスのpinch zoomを抑止する
emoji: 🤏
slug: 2019-11-03-react-hooks-pinch-zoom
type: tech
topics:
  - reacthooks
  - react
  - javascript
published: true
terrier_date: '2019-11-03T13:45:05.900Z'
terrier_export_from: 'https://blog.terrier.dev/blog/2019/20191103224505-react-hooks-pinch-zoom'
---

> 記事作成日: 2019/11/03

あまり推奨されるものではないのだが、どうしてもpinchによるzoomを抑制したいケースはまれにある。
しかし久々にこれをやろうとしたら`<meta>`タグの`user-scroll=no`ではダメになってたので覚え書き。

* https://developer.mozilla.org/ja/docs/Web/API/EventTarget/addEventListener
* https://qiita.com/yukiTTT/items/773356c2483b96c9d4e0
* http://iphone.f-tools.net/html5/Kakudai-Kinsi.html

この辺を参考にしながら、hooksにすると下記のような具合になった。

```js
const useDisablePinchZoomEffect = () => {
  useEffect(() => {
    const disablePinchZoom = (e) => {
      if (e.touches.length > 1) {
        e.preventDefault()
      }
    }
    document.addEventListener("touchmove", disablePinchZoom, { passive: false })
    return () => {
      document.removeEventListener("touchmove", disablePinchZoom)
    }
  }, [])
}

```

`document`でなく一部の要素を抑止したいならこんな感じだろう。

```js

const SuspendPinchZoom = ({ children }) => {
  const ref = useRef<HTMLDivElement>(null)
  useLayoutEffect(() => {
    const target = ref.current
    if (!target) return
    const disablePinchZoom = (e) => {
      if (e.touches.length > 1) {
        e.preventDefault()
      }
    }
    target.addEventListener("touchmove", disablePinchZoom, { passive: false })
    return () => {
      target.removeEventListener("touchmove", disablePinchZoom)
    }
  }, [])
  return <div ref={ref}>{children}</div>
}
```
