---
title: React Hooksからpng-pongやCanvas経由でBitmapを直描画する
emoji: ☄️
type: tech
topics:
  - ReactHooks
  - React
  - png
published: true
terrier_date: '2019-10-14T08:03:57.157Z'
terrier_export_from: 'https://blog.terrier.dev/blog/2019/20191014170357-png-pong-draw-image'
---

> 記事作成日: 2019/10/14

[png-pong](https://github.com/gdnmobilelab/png-pong)でReact Hooksを通して画像を描画出来たメモ。

### デモ

https://stackblitz.com/edit/react-ts-yaiy4a?embed=1&hideExplorer=1&ctl=1

```jsx

const Img = styled.div<{
  size: number
  bg: string
}>(({ size, bg }) => ({
  width: `${size}px`,
  height: `${size}px`,
  backgroundImage: `url(${bg})`,
  filter: "blur(0.2px)"
}))

const bitsToStr = (bits: number[][][]) => {
  const width = bits[0].length
  const height = bits.length
  const bit = new Uint8ClampedArray(bits.flat(2))
  const buffer = createFromRGBAArray(width, height, bit)
  const b642 = base64.fromByteArray(new Uint8Array(buffer))

  return "data:image/png;base64," + b642
}


const noise = (width, height, color: Color = [0, 0, 0]) => {
  return Array(width)
    .fill(0)
    .map((_, x) => {
      return Array(height)
        .fill(0)
        .map((_, y) => Math.random() > 0.5)
        .map((b) => (b ? [0, 0, 0, 255] : [255, 255, 255, 255]))
    })
}

const NoiseImage = () => {
  const [bgurl, setBgurl] = useState<string>("")
  const width = 100
  const height = 100

  useEffect(() => {
    const bitmap = noise(width, height, 5)
    setBgurl(bitsToStr(bitmap))
  }, [])
  return <Img size={100} bg={bgurl}></Img>
}

const App = () => {
  return (
    <div>
      <NoiseImage />
    </div>
  )
}
```

### 概要

こんな感じでeffectを利用しているが、正直別にreactの機能そんなに使ってない。

```jsx
const Img = styled.div<{
  size: number
  bg: string
}>(({ size, bg }) => ({
  width: `${size}px`,
  height: `${size}px`,
  backgroundImage: `url(${bg})`,
  filter: "blur(0.2px)"
}))

const NoiseImage = () => {
  const [bgurl, setBgurl] = useState<string>("")
  const width = 100
  const height = 100

  useEffect(() => {
    const bitmap = noise(width, height, 5)
    setBgurl(bitsToStr(bitmap))
  }, [])
  return <Img size={100} bg={bgurl}></Img>
}
```

生成したビットマップを下記のように`createFromRGBAArray`で渡してる。
その結果をbase64のURLとして保持し、あとはそのまま`background-image`としてstyled-componentsに流し込んでる
このへんは完全にpng-pongのおかげだ。

```ts
const bitsToStr = (bits: number[][][]) => {
  const width = bits[0].length
  const height = bits.length
  const bit = new Uint8ClampedArray(bits.flat(2))
  const buffer = createFromRGBAArray(width, height, bit)
  const b642 = base64.fromByteArray(new Uint8Array(buffer))

  return "data:image/png;base64," + b642
}
```

これだけだと正直Canvasの方が良いが、pngが使えることで`background-clip: text`の背景に使うことなども出来る。

https://stackblitz.com/edit/react-ts-egg6dg?embed=1&file=index.tsx&hideExplorer=1&view=preview&ctl=1

今の所Canvasに勝る利点はなさそうではあるが、bitmap部分が複雑でWASMを利用するなどでCanvasよりも素早く描画出来るなどの可能性がありそう

## Canvasだけでやる

ここまでライブラリでやってみたが、Canvasだけでも可能だ。

### demo 
https://stackblitz.com/edit/react-ts-x3yyna?ctl=1&embed=1&file=index.tsx

### 解説
肝になってるのはこのへん。canvasは隠しつつ、imageとして吐き出している。
ただ当然canvasの上限サイズを越えてたりするとimgの変換から切れるので注意が必要

```jsx

const CloakCanvas = styled.canvas`
  display: none;
`

const NoiseCanvas: FC<{ bitmap: number[] }> = ({ bitmap }) => {
  const ref = useRef<HTMLCanvasElement>()
  const [url, setUrl] = useState()
  useLayoutEffect(() => {
    if (ref.current === undefined) {
      return
    }
    const ctx = ref.current.getContext("2d")
    if (ctx === null) {
      return
    }
    const img = ctx.createImageData(100, 100)
    bitmap.forEach((c, i) => {
      img.data[i] = c
    })
    ctx.putImageData(img, 0, 0)
    const imgUrl = ref.current.toDataURL("image/png")
    setUrl(imgUrl)
  }, [bitmap])
  return (
    <div>
      <CloakCanvas ref={ref}></CloakCanvas>
      <img src={url} />
    </div>
  )
}


const App = () => {
  const c = noiseBitmap()
    .map(({ color }) => color)
    .flat()

  return (
    <div>
      <NoiseCanvas bitmap={c}></NoiseCanvas>
    </div>
  )
}
```
