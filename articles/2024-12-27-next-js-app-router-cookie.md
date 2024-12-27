---
title: next.jsのAppRouterでCookieをいい感じで扱う落とし所を考える
emoji: 🍪
type: tech
topics:
  - nextjs
  - typescript
  - swr
  - react
published: true
---

next.jsでCookieをサーバーでもクライアントでもいい感じに扱うにはどうするか考えた。
今のところこのあたりが落とし所かな〜？ぐらいな温度感なので、特にベストプラクティスとかではない。

全体感を俯瞰したい場合は下記でざっくり参照可能
* https://codesandbox.io/p/devbox/7d8ts6

# Layout
今回サーバーからの初回レンダリングも考慮したいケースを考えるので、サーバーからまずCookieを引っ張ってくる。これをContextに詰め込んでクライアント側で利用できるようにする方向で考えた


まず全体で利用できるようにlayoutで取得する

```tsx
// layout.tsx
"use server"
import { ServerCookie } from "./ServerCookie"

export default async function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="ja">
      <body>
        <ServerCookie>
          {children}
        </ServerCookie>
      </body>
    </html>
  )
}
```
## Server ComponentからCookieを取得する
実際にcookieを取得するServer Component

```tsx
// ServerCookie.tsx
"use server"
import { cookies } from "next/headers"
import { FC, PropsWithChildren } from "react"
import { CookieProvider, CookieValue } from "./CookieContext"

const getServerCookies = async () => {
  const resolvedCookie = await cookies()
  return Object.fromEntries(resolvedCookie.getAll().values().map((cookieValue) => {
    return [cookieValue.name, cookieValue.value]
  }))
}

export const ServerCookie: FC<PropsWithChildren> = async ({ children }) => {
  const value = await getServerCookies()
  return <CookieProvider value={value}>
    {children}
  </CookieProvider>
}
```

## Contextに詰め込む

ContextとProviderは下記のようになる

```tsx
// ./CookieContext.tsx
"use client"
import { createContext, FC, PropsWithChildren } from "react"

export type CookieValue = Partial<Record<string, string>>

export const CookieContext = createContext<CookieValue>({})

export const CookieProvider: FC<PropsWithChildren<{ value: CookieValue }>> = ({ value, children }) => {
  return <CookieContext.Provider value={value}>
    {children}
  </CookieContext.Provider>
}
```

# Client側

## useSyncExternalStoreで読み取り側の作成

ここまでにContextに入れているのはあくまでサーバーで取得した値なので、書き換え等を考慮していない。
これを取り扱うために`useSyncExternalStore`を使い、書き込み時の変更を考慮する。

また、ここではCookieの扱いの簡略化のために[`cookies-next`](https://www.npmjs.com/package/cookies-next)を利用している。ここは各々好きなものを使うと良い

```tsx
// useCookieValue.ts
import type { SerializeOptions } from "cookie"
import { getCookie, setCookie, deleteCookie } from "cookies-next/client"
import { useContext, useSyncExternalStore } from "react"
import { CookieContext } from "./CookieContext"

type Listener = () => void

// hooksが複数回呼び出されても有効なように、listenerはglobalで管理
const globalCookieListeners = new Set<Listener>()

export function useCookieStore(key: string) {
  const serverCookie = useContext(CookieContext)

  const value = useSyncExternalStore<string | undefined>((listener: Listener) => {
    globalCookieListeners.add(listener)
    return () => globalCookieListeners.delete(listener)
  }, () => getCookie(key), () => serverCookie[key])

  return {
    getValue: () => {
      return value
    },
    setValue: (value?: string, options?: SerializeOptions) => {
      setCookie(key, value, options ?? {})
      globalCookieListeners.forEach(l => l())
    },
    deleteValue: () => {
      deleteCookie(key)
      globalCookieListeners.forEach(l => l())
    }
  }
}
```

### 番外編:useSWRで代替する

`useSyncExternalStore`にこだわりがなければ`useSWR`などを利用して下記のようにすることも考えられる

```ts
import { parse } from "cookie"
import { useContext } from "react"
import useSWR from "swr"
import { CookieContext } from "./CookieContext"

export function useSWRCookie() {
  const serverCookie = useContext(CookieContext)
  return useSWR(["cookieValue"], () => {
    return parse(document.cookie)
  }, {
    fallbackData: serverCookie
  })
}
```

## コンポーネントで利用する

コンポーネント側では下記のように利用する。

```tsx
// Counter.tsx
"use client"
import { counterDeserlizer, counterSerializer } from "./cookieCounterConverter"
import { useCookieStore } from "./useCookieValue"

export const Counter = () => {

  const targetKey = "counter"
  const cookieCounter = useCookieStore(targetKey)

  const value = counterDeserlizer(cookieCounter.getValue())
  const onUpdate = (value: number) => {
    cookieCounter.setValue(counterSerializer(value))
  }
  return <div>
    <div>
      v:{value}
    </div>
    <button onClick={() => onUpdate(value + 1)}>
      +
    </button>
    <button onClick={() => onUpdate(value - 1)}>
      -
    </button>
  </div>
}
```

`cookieCounterConverter`は下記のようにstringから変換しているだけの関数。
今回の例だとちょっと過剰気味だが、もう少し複雑なケースや複数箇所で利用する場合だとこのあたりが欲しくなってくる

```tsx
// cookieCounterConverter.ts
export const counterDeserlizer = (value?: string): number => {
  return value ? parseInt(value) : 0
}
export const counterSerializer = (value: number): string => {
  return value.toString()
}
```

# APIでも利用する

APIから利用する場合の例。converterを作っておくと、ここでも利用ができる

```ts
import { cookies } from "next/headers"
import { counterSerializer, counterDeserlizer } from "../../../cookieCounterConverter"
import { NextResponse } from "next/server"

export const GET = async () => {
  const cookieValues = await cookies()
  const targetKey = "counter"

  // const cookies = await getServerCookies()
  const value = counterDeserlizer(cookieValues.get(targetKey)?.value)
  cookieValues.set(targetKey, counterSerializer(value + 1))
}
```

ひとまずここまででサーバー・クライアントである程度共通に扱えるようなものが出来た。`cookies-next`はserver/cookieどちらでも扱えるのだが、微妙に同名なわりに引数や挙動が変わるので、好みが分かれそうだ。
cookie自体に対してはあまり複雑な構造化はさせずstringで受け渡しをして、外側でコンバーターのようなものを挟んでやりとりする方がいっそスッキリしそうだった