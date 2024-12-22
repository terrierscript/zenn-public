---
title: ReactでAnimationPresenceみたいなのをライブラリ無しで行う
emoji: 🎁
type: tech
topics:
  - react
  - framermotion
published: true
---

Reactで要素の表示非表示のアニメーションをさせる際、表示非表示をフラグで子要素を利用したいことがある。

[motion](https://motion.dev/docs/react-animate-presence) (framer-motion)のAnimationPresenceなどを利用すればこれは解決できる。

```tsx
const Item = (show) => {
  return <AnimationPresence>
    {show && <Box>Hello</Box>}
  </AnimationPresence>
}
```

今回motionを避けたかったので自前で`useRef`で実装を試みた。

## `useRef`で実装

アニメーション自体は標準の[Web Animations API](https://developer.mozilla.org/ja/docs/Web/API/Web_Animations_API)を使った場合、こんな具合になる

```tsx

const MyAnimatePresence: FC<PropsWithChildren<{}>> = ({ children }) => {
  const animateRef = useRef(children)
  const show = !!children
  const keyframes = [{ opacity: 0, }, { opacity: 1, }]
  const hiddenProps = show ? keyframes[1] : keyframes[0]
  const ref = useRef<HTMLDivElement>(null)
  useEffect(() => {
    animateRef.current = children // 1
    const direction = show
      ? "normal"
      : "reverse"

    ref.current?.animate(keyframes, {
      fill: "both",
      easing: "ease-in",
      direction,
      duration: 500
    })
  }, [children, show])
  // 2
  return <div ref={ref} style={hiddenProps}>
    {children || animateRef.current}
  </div>
}

```

仕組みとしては下記のような具合になっている
1. animateRefなどでアニメーション前の要素を保持
2. 表示部分で前の要素が残っていればそれを表示する

これで簡易的に再現することができる。
複数要素に対応してなかったりするので、そういう要件の場合はおとなしくmotionを使うのが良いかもしれない。

## 番外編: Mantineの`<Transition>`と組み合わせる
Mantineの`<Transition>`と組み合わせる場合は下記のような具合になる

```tsx
const MyAnimatePresence: FC<PropsWithChildren<{}>> = ({ children }) => {
  const animateRef = useRef(children)
  const show = !!children
  return <Transition
    mounted={show}
    transition="fade"
    duration={120}
    timingFunction="ease"
    onExit={() => { animateRef.current = children }}
  >{(styles) => {
    return <div style={styles}>
      {children || animateRef.current}
    </div>
  }}
  </Transition>
}
```