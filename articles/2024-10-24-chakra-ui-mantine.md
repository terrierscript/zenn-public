---
title: Chakra UIからmantineに置き換える準備をする
emoji: 🕊️
type: tech
topics:
  - chakraui
  - mantine
  - typescript
  - react
published: true
---

[Chakra UIのv3がリリース](https://www.chakra-ui.com/blog/00-announcing-v3)され、スニペットをデフォルトとするなどかなり使い勝手が変わった。

個人的にv3以降Chakra UIから恩恵を得られない可能性が高まってきたので、他のライブラリとの共存、移植の方法を試したい。

今回、[mantine](https://mantine.dev/)を移植先として、どちらにも存在する`Popover`のコンポーネントを移植する場合の手順を記載する。

[npm trends](https://npmtrends.com/@chakra-ui/react-vs-@mantine/core)で比較的利用数が近く、使い勝手も近いmantineを選択したが、他のライブラリでもほとんど同じような手順になるだろう

また、今回は「それなりに機能が近い」ぐらいを目標として、見た目が完全に一致することは求めないレベルで考える

# 前提: Chakra UIの場合

まず元となるChakra UIでのPopoverはこのような具合になる

```tsx
import { FC } from "react"
import { Box, Button, Popover, PopoverArrow, PopoverBody, PopoverCloseButton, PopoverContent, PopoverHeader, PopoverTrigger } from "@chakra-ui/react"

const PopoverSample = () => {
  return <Popover>
    <PopoverTrigger>
      <Button>Trigger</Button>
    </PopoverTrigger>
    <PopoverContent>
      <PopoverArrow />
      <PopoverCloseButton />
      <PopoverHeader>ヘッダーだよ</PopoverHeader>
      <PopoverBody>Bodyだよ</PopoverBody>
    </PopoverContent>
  </Popover>
}
```

# 1. mantineを共存させる

まずはコンポーネントのインストール

```
$ yarn add @mantine/core @mantine/hooks
$ yarn add --dev postcss postcss-preset-mantine postcss-simple-vars
```

次にレイアウトに変更を加える。CSSの読み込みと、MantineProviderの追加を行う。`ChakraProvider`の付近が良いだろう

```tsx
// app/layout.tsx
import '@mantine/core/styles.css'; // 追加

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="ja" suppressHydrationWarning>
      <head>
        <ColorSchemeScript />
      </head>
      <body>
        <ChakraProvider>
          <MantineProvider>
            {children}
          </MantineProvider>
        </ChakraProvider>
      </body>
    </html>
  )
}
```

また、[next app用のテンプレート](https://github.com/mantinedev/next-app-min-template/blob/master/app/layout.tsx)を参考に、`<ColorSchemaScript />`と`suppressHydrationWarning`を追加している。


# 2. Popoverを移植してみる

Popoverを最低限置き換えるとこのような形になる。

```tsx
import { Box, Stack, Button } from "@chakra-ui/react"
import { Divider, Popover } from "@mantine/core"

const PopoverSample = () => {
  return <Popover withArrow radius={"md"}>
    <Popover.Target>
      <Button>Trigger</Button>
    </Popover.Target>
    <Popover.Dropdown>
      <Stack>
        <Box>ヘッダーだよ</Box>
        <Divider />
        <Box>Bodyだよ</Box>
      </Stack>
    </Popover.Dropdown>
  </Popover>
}
```

あえてChakra UIとmantieを混ぜている。気持ち悪さはあるものの、特に競合することもなく動いてくれるのは移植においてはありがたいことだろう。

## 2-2. Close Buttonもほしい場合
Close Buttonはmantineには無いので、そこまで欲しい場合は自前する必要がある。
（こちらはすべてmantineの場合で記載している）

```tsx
import { Box, Button, CloseButton, Divider, Group, Popover, Stack } from "@mantine/core"
import { useState } from "react"

export const PopoverSample2 = () => {
  const [opened, setOpened] = useState(false)
  return <Popover withArrow radius={"md"} opened={opened} onChange={setOpened}>
    <Popover.Target >
      <Button onClick={() => setOpened((o) => !o)}>Trigger</Button>
    </Popover.Target>
    <Popover.Dropdown>
      <Stack>
        <Group>
          <Box>ヘッダーだよ</Box>
          <CloseButton onClick={() => setOpened((o) => !o)} />
        </Group>
        <Divider />
        <Box>Bodyだよ</Box>
      </Stack>
    </Popover.Dropdown>
  </Popover>
}
```

# おまけ: Chakra側の利用を塞ぐ

今後うっかりChakra UI側のPopoverを使わないように、TypeScriptの定義でPopoverを塞いでおくと便利だ。

```ts
// chakra-ui-overwrite.d.ts
export * from "@chakra-ui/react"

module "@chakra-ui/react" {
  /** @deprecated mantineを使いましょう */
  const Popover: never

  // 他のコンポーネントも塞いでおきたいなら下記も追加
  // const PopoverArrow: never
  // const PopoverBody: never
  // const PopoverCloseButton: never
  // const PopoverContent: never
  // const PopoverHeader: never
  // const PopoverTrigger: never
}
```

neverまではしたくない場合は下記のようにしてdeprecatedだけわかるようにするのでも良いだろう

```ts
export * as Chakra from "@chakra-ui/react"

module "@chakra-ui/react" {
  /** @deprecated mantineを使いましょう */
  const Popover: Chakra.Popover
}
```

# まとめ
このようにChakra UIとmantineの混在する方法を示した。CSSの頃のように競合せずに順次置き換えていけることはありがたい。（とはいえエッジケースやdark modeのようなものを含む場合は未検証なので、そのような場合は要注意）

Chakra UI v2 から v3へ一気に移植するのもかなり課題が大きい可能性がある[^1]ので、`Chakra UI v2` -> `別なライブラリ` -> `Chakra UI v3`のような置き換えも検討の一つになるだろう

[^1]: migration guideはあるものの、例えばButtonの`colorSchema`が`colorPalette`になっており、更にこれがTypeScriptとしてはエラーとして検出もされないなど、カロリーが高そうだった。