---
title: Chakra UIにCSSのscroll-snapだけのライブラリなしカルーセルを実装する
emoji: 🎠
type: tech
topics:
  - chakraui
  - css
  - html
  - typescript
published: true
---

Chakra UIにカルーセルを組み込もうとすると、うまく動かなかったり色々面倒だった。
ちょっとしたカルーセルであればCSSの`scroll-snap-`系のプロパティを使うことで解決できるので、これを組み込むことを考えた。

なお、今回画像は[dog.ceo](https://dog.ceo/)を利用させていただきた。

## まずCSSだとどうなるのか？

元となるCSSでの実装はこのような具合になる。

```css
.slider{
  width: 200px;
  height: 200px;
  overflow-x: scroll;
  align-items: center;
  flex-direction: row;
  scroll-snap-type: x mandatory;  
  display: flex;
}
.slide-img{
  width:  200px;
  height: 200px;
  object-fit: cover;
  scroll-snap-align: center;
  scroll-snap-stop: always;
  aspect-ratio: 1;
}
```
```html
<div class="slider">
  <img class="slide-img" src="https://images.dog.ceo/breeds/dhole/n02115913_3753.jpg">
  <img class="slide-img" src="https://images.dog.ceo/breeds/corgi-cardigan/n02113186_12793.jpg">
  <img class="slide-img" src="https://images.dog.ceo/breeds/pointer-germanlonghair/hans2.jpg">
</div>
```

仕組みとしては親コンテナに`scroll-snap-type: x mandatory`をつけて、子の要素に`scroll-snap-stop: always`をつける。ついでに`scroll-snap-align: center`も指定している。

これで下記のように簡単なカルーセルができた。

![](https://storage.googleapis.com/zenn-user-upload/9d3e336a2d03-20230228.gif)

CSSのみで実装する方法は色々と紹介されているので、更に知りたい人は「CSS only carousel」などで検索すると良いだろう

## Chakra UIに組み込んでいく

これをChakra UIに組み込むと、下記のようになる。
```tsx
import { Image, Box, AspectRatio, HStack } from "@chakra-ui/react"
import { FC } from "react"

export const PureCarousel: FC<{ images: string[] }> = ({ images }) => {
  const size = 200
  return <Box>
    <HStack overflowX="scroll"
      w={size}
      sx={{
        scrollSnapType: "x mandatory",
      }}>
      {images.map((img, i) => {
        return <AspectRatio ratio={1} minW={size} sx={{
          scrollSnapAlign: "center",
          scrollSnapStop: "always"
        }}>
          <Image key={i} src={img} w={size} draggable={false} userSelect="none" />
        </AspectRatio>
      })}
    </HStack>
  </Box>
}
```
`sx`を利用して`scrollSnapType`のように記載していく。これで同様のカルーセルができる。

## 応用編: 左右ボタンをつける

ちょっとだけ応用させて左右に移動できるボタンぐらいつけてみる。
スクロールするコンテナに`ref`を設定し、これをボタンが押されたタイミングで左右に移動させる。

```tsx
export const PureCarousel: FC<{ images: string[] }> = ({ images }) => {
  const ref = useRef<HTMLDivElement>(null)
  const size = 200
  return <HStack alignItems={"stretch"}>
    <HStack bg="gray.100" w={6} justifyContent="center" cursor={"pointer"} onClick={() => {
      if (!ref.current) return
      console.log(ref.current.scrollLeft)
      ref.current.scrollTo({
        left: ref.current.scrollLeft - size,
      })
      // scrollToPosition(currentPosition - 1)
    }}>
      <Box>◀️</Box>
    </HStack>
    <HStack
      scrollBehavior={"smooth"}
      overflowX="scroll"
      ref={ref}
      w={size}
      sx={{
        scrollSnapType: "x mandatory",
      }}>

      {images.map((img, i) => {
        return <AspectRatio ratio={1} minW={size} sx={{
          scrollSnapAlign: "center",
          scrollSnapStop: "always"
        }}>
          <Image key={i} src={img} w={size} draggable={false} userSelect="none" />
        </AspectRatio>
      })}

    </HStack>
    <HStack bg="gray.100" w={6} justifyContent="center" cursor={"pointer"}
      onClick={() => {
        if (!ref.current) return
        ref.current.scrollTo({
          left: ref.current.scrollLeft + size,
        })
      }}>
      <Box>
        ▶️
      </Box>
    </HStack>
  </HStack>
```
これで下記のように移動ボタンができた。

![](https://storage.googleapis.com/zenn-user-upload/e89443f183cb-20230228.gif)

