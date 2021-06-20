---
title: Chakra UIの再利用コンポーネント拡張の方法あれこれ
emoji: 🎊
type: tech
topics:
  - chakraui
  - typescript
  - react
  - javascript
published: true
---

Chakra UIでカスタムコンポーネントを作ろうとした時、いくつかやり方があることがわかってきた。

## 1. 別コンポーネントとして普通にラッパーを作る

一番驚きの少ない方法

```tsx
const RoundedOutlineButton: FC<ButtonProps> = (props) => {
  return <Button
    variant="outline"
    borderStyle="solid"
    borderWidth="2px"
    rounded="full"
    px={10}
    py={5}
    letterSpacing="0.1em"
    {...props} />
}
```

ほぼ8割はこれで解決出来る。

```tsx
<RoundedOutlineButton>hello</RoundedOutlineButton>

// 更にもうちょっと拡張したいなら
<RoundedOutlineButton p={10}>hello</RoundedOutlineButton>
```


* 利点
    * 最もお手軽
    * 他への影響が無い
* 欠点
    * `refs`が絡んでる際に一工夫必要になる

### 欠点: refsが絡んだ場合の問題

例えば上記で言えば`PopOver`と組み合わせた場合、refsの問題が起きる

```tsx
<Popover>
  <PopoverTrigger>
    <RoundedOutlineButton>
  </PopoverTrigger>
  <PopoverContent>
    <PopoverHeader>Confirmation!</PopoverHeader>
    <PopoverBody>Are you sure you want to have that milkshake?</PopoverBody>
  </PopoverContent>
</Popover >

```

```
Warning: Function components cannot be given refs. 
Attempts to access this ref will fail. Did you mean to use React.forwardRef()?
```

ということで、refが使われるようなものは`forwardRef`を使う必要がある

```tsx
import { forwardRef } from '@chakra-ui/react'

// before
// const RoundedOutlineButton: FC<ButtonProps> = (props) => { 

// after
const RoundedOutlineButton = forwardRef<ButtonProps, "button">((props, ref) => { 
  return <Button
    variant="outline"
    borderStyle="solid"
    borderWidth="2px"
    rounded="full"
    ref={ref}
    px={10}
    py={5}
    letterSpacing="0.1em"
    {...props} />
})
```

`React.forwardRef`でも良いが、型をつけられている[`chakra-ui`の`forwardRef`](https://github.com/chakra-ui/chakra-ui/blob/98efb69699b5cfc47158947ed82ea534b67555c1/packages/system/src/forward-ref.tsx#L8)を使うとTSだとちょっとだけ嬉しいかもしれない



## 2. Themeで全体を変えてしまう

もし「このサイトは全部スタイルを変えたい・変えれる」という要件が満たせる場合なら`theme`を使う手法もある

```tsx
import { extendTheme } from '@chakra-ui/react'

const providerTheme = extendTheme({
  components: {
    Button: {
      variants: {
        outline: {
          bg: "white",
          borderStyle: "solid",
          borderWidth: "2px",
          rounded: "full",
          px: 10,
          py: 5,
          letterSpacing: "0.1em",
        },
      }
    }
  }
})
```

```tsx
<ChakraProvider theme={providerTheme}>
  <Button variant="outline" colorScheme="blue">Provider Theme</Button>
</ChakraProvider>
```

* 利点
    * コンポーネントを利用したらすべて変えれる
    * refs周りで困らなくて済む
* 欠点
    * グローバルなので、他への影響が無いか気をつける必要がある
    * themeを管理しきる一種の覚悟が必要

見て分かる通り、ほとんど生CSSを書いてるに近くなるので、かなり諸刃の剣となる手法。


## 3. Themeで新しいvariantとして作る

2をもうちょっとだけマイルドにしたもの。新しいvariantを生やす

```tsx
import { theme, extendTheme } from '@chakra-ui/react'

const providerThemeAppendVariant = extendTheme({
  components: {
    Button: {
      variants: {
        customOutline: (props) => {
          return {
            ...theme.components.Button.variants.outline(props),
            bg: "white",
            borderStyle: "solid",
            borderWidth: "2px",
            rounded: "full",
            px: 10,
            py: 5,
            letterSpacing: "0.1em",
          }
        },
      }
    }
  }
})
```

```tsx
<ChakraProvider theme={providerThemeAppendVariant}>
  <Button variant="customOutline" colorScheme="blue">Provider Custom variant Theme</Button>
</ChakraProvider>
```

* 利点
    * 既存への影響は無くせる
    * refs周りで困らなくて済む
* 欠点
    * やりたいことの割には大仰になりがちで、変更範囲も大きそう
    * 型周りが若干気を使う可能性がある

## 4. `chakra` factoryの機能で拡張する

かなり限定的なのだが、[factory](https://chakra-ui.com/docs/features/chakra-factory)での拡張も考えられる。

```tsx
const RoundedOutlineButton = chakra(Button, {
  baseStyle: {
    bg: "white",
    borderStyle: "solid",
    borderWidth: "2px",
    rounded: "full",
    px: 10,
    py: 5,
    letterSpacing: "0.1em",
  }
})
```

```jsx
<RoundedOutlineButton variant="outline" colorScheme="blue">
  Factory Custom Button
</RoundedOutlineButton>
```

* 利点
    * refs周りで困らなくて済む
* 欠点
    * ~~propsが取れないので拡張性が薄い~~
    * 既存のvariantとの重ね合わせみたいになって、脳みそがパンクしそう

~~propsが取れないのはかなり致命的な部分なので、この手法はほとんど使えない可能性がある。~~

### 2021/06/20追記

v1.7.0より、FactoryのbaseStyleも関数を与えることでpropsを取得できるようになった[^1]

```tsx
const RoundedOutlineButton = chakra(Button, {
  baseStyle: (props) => {
    return {
      bg: props.bg,
      borderWidth: "2px",
      rounded: "full",
      px: 10,
      py: 5,
      letterSpacing: "0.1em",
    }
  }
})

```

themeも取得できるので、このようなことも可能

```tsx
const RoundedOutlineButton = chakra(Button, {
  baseStyle: ({ theme, ...props }) => {
    return {
      ...theme.components.Button.variants.outline(props),
      // borderStyle: "solid",
      borderWidth: "2px",
      rounded: "full",
      px: 10,
      py: 5,
      letterSpacing: "0.1em",
    }
  }
})
```

また、型がうまく対応してないものの、下記のようにCSS以外のpropsも渡されているので`colorScheme`のようなものも利用は可能

```tsx
const RoundedOutlineButton = chakra(Button, {
  baseStyle: (props) => {
    // @ts-ignore
    const { colorScheme } = props
    return {
      bg: `white`,
      color: `${colorScheme}.600`,
      borderColor: `${colorScheme}.600`,
      borderStyle: "solid",
      borderWidth: "2px",
      rounded: "full",
      px: 10,
      py: 5,
      letterSpacing: "0.1em",
      _hover: {
        bg: `${colorScheme}.50`
      }
    }
  }
})
```

[^1]: PR投げたら通りました。[https://github.com/chakra-ui/chakra-ui/releases/tag/%40chakra-ui%2Fsystem%401.7.0](@chakra-ui/system@1.7.0)

## 結論

基本的には1で、forwardRefに気をつけていくのが良さそう。
覚悟が出来るならthemeも検討してよいかも