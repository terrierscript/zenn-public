---
title: next.jsでモノレポツールを入れずに複数のサービス起動出来るようにしたい
emoji: 👨‍👩‍👧‍👦
type: tech
topics:
  - typescript
  - nextjs
  - javascript
published: true
---

next.jsで同じリポジトリを使いつつ複数の別なアプリケーションとして起動したいケースが度々有る。

この場合、殆どはturbolinkやnxなどのmonorepoツールを利用する方法が広く知られている。
next.jsの公式でも[Multi zoneの例](https://nextjs.org/docs/advanced-features/multi-zones)ではmonorepoが利用されている

一方でmonorepoは各所対応してない場合があったり気を使う部分も多く、一定の覚悟を必要になる

今回モノレポツールを使わない、もう少しライトな方法を2つ思いついたのでそれぞれまとめていきたい

1. ディレクトリごとにアプリを切り分け、ENVからの指定で切り替える
2. webpackの設定を調整して依存関係を解決させる

# 1. ディレクトリごとにアプリを切り分け、ENVからの指定で切り替える

こちらはnextのレールからほぼ外れないのでそれほど覚悟を必要としないやり方。
デメリットとしてURLにディレクトリが含まれたりするので見栄えとしてあまり良くない点がある。

この方法では、下記のようなディレクトリ構成を取る。

```
src
└── pages
    ├── _app.tsx
    ├── index.tsx  // リダイレクトするだけ
    ├── mainapp　  // メインアプリ
    │   └── index.tsx
    └── subapp     // サブアプリ
        └── index.tsx
```

ルートレベルにはindexのみ配置し、それぞれアプリごとで分離する。
mainappをルートに配置するのも可能ではあるが、管理の面を考えるとディレクトリを分けるほうが無難なため今回はこの方式で進める。
サブアプリはローカルでのみ起動できれば良いというケースもあると思うので、その場合はmainappをディレクトリに閉じ込める必要も無いだろう


## package.jsonにコマンド追加

まずはpackage.jsonにサブアプリ起動用のコマンドを追加する。

```json
    "dev": "next dev",
    "dev:subapp": "APP_MODE=SUBAPP next dev -p 3002",
```

`start`や`build`は同様だがここでは省略する

## next.configで切り分けれるようにする

次にnext.configでenvによって切り替える部分を設定する。

```js
// envごとに切り替える設定
const appendConfig = (appMode) => {
  switch (appMode) {
    case "SUBAPP":
      return {
        // distDirが同じだと、ローカルで起動している際に壊れる
        distDir: ".next-subapp", 
        redirects: async () => ([{
          source: "/mainapp/:path*",
          destination: "/subapp",
          permanent: false,
        }, {
          source: "/api/mainapp/:path*",
          destination: "/error",
          permanent: false,
        }])
      }
    default:
      return {
        redirects: async () => ([{
          source: "/subapp/:path*",
          destination: "/mainapp",
          permanent: false,
        }, {
          source: "/api/subapp/:path*",
          destination: "/error",
          permanent: false,
        }])
      }
  }
}

// 共通のconfigの例。
const baseAppConfig = {
  pageExtensions: ["tsx","ts", "page.tsx", "page.ts"]
}

module.exports = () => {
  const appMode = process.env.APP_MODE ?? ""
  const config = appendConfig(appMode)
  return {
    ...baseAppConfig,
    ...config
  }
}
```

### middlewareでやるなら？
next 12.2以上であれば、redirectの設定は`middleware.ts`でも可能だ

```ts
import { NextRequest, NextResponse } from "next/server"

export function middleware(request: NextRequest) {
  if (process.env.APP_MODE === "subapp") {
    if (request.nextUrl.pathname.startsWith("/mainapp") || request.nextUrl.pathname.startsWith("/api/mainapp")) {
      return NextResponse.redirect(new URL('/subapp', request.url))
    }
  }
  if (request.nextUrl.pathname.startsWith("/subapp") || request.nextUrl.pathname.startsWith("/api/subapp")) {
    return NextResponse.redirect(new URL('/mainapp', request.url))
  }
  return NextResponse.next()
}

export const config = {
  matcher: [
    "/mainapp/:path*",
    "/app/mainapp/:path*",
    "/subapp/:path*",
    "/app/subapp/:path*"
  ]
}
```

## 更に細かいところ
`next.config.js`への設定までで、本題の切り分けとしては完了した。
ここからはもう少し踏み込んだ部分を記述していく

### Layoutを分離する


レイアウトの部分も切り替える。
レイアウトは`useRouter`のパスで切り替えを行う

```tsx
// _app.tsx
const SubappLayout: FC<PropsWithChildren<{}>> = ({ children }) => {
  return <Stack>
    <Box bg="blue.100">Sub app</Box>
    <Box>
      {children}
    </Box>
  </Stack>
}

const MainappLayout: FC<PropsWithChildren<{}>> = ({ children }) => {
  return <Stack>
    <Box bg="red.100">Main app</Box>
    <Box>
      {children}
    </Box>
  </Stack>
}

const AppLayout: FC<PropsWithChildren<{}>> = ({ children }) => {
  const router = useRouter()
  if (router.pathname.startsWith("/subapp")) {
    return <SubappLayout>
      {children}
    </SubappLayout>
  }

  return <MainappLayout>
    {children}
  </MainappLayout>

}

function App({ Component, pageProps }: AppProps) {
  return <AppLayout>
    <Component {...pageProps} />
  </AppLayout>
}

```

### indexにリダイレクト設定

好みにはなるが、トップルートの`index.ts`にリダイレクトをかけたい場合は`getServerSideProps`などで`env`を見る形にすると良いだろう

```tsx
import { GetServerSideProps } from 'next'

export const getServerSideProps: GetServerSideProps = async () => {
  if (process.env.APP_MODE === "subapp") {
    return {
      redirect: {
        destination: "/subapp",
        statusCode: 301
      }
    }
  }
  return {
    redirect: {
      destination: "/mainapp",
      statusCode: 301
    }
  }
}

export default function Home() {
  return null
}
```


# 2. webpackの設定を調整して依存関係を解決させる

こちらは比較的きれいな形で実装できるが、webpack部分を変更する必要が出うるので少し覚悟が必要な方法。

こちらの方法では、ルートディレクトリごと切り分ける

```
.
├── app-mainapp // メインアプリ
│   ├── next-env.d.ts
│   ├── pages
│   │   └── index.tsx
│   └── tsconfig.json
├── app-subapp // サブアプリ
│   ├── next-env.d.ts
│   ├── pages
│   │   └── index.tsx
│   └── tsconfig.json
├── shared // 共通部分
│   └── SharedComponent.tsx
│
```

こちらの方法ではほぼ完全にそれぞれのnext.jsは独立するため、Layout等に処理を施す必要がなくなる。
ただ、共通する部分についてまとめている`shared`については普通に動かすと共通部分はコケてしまうので対応が必要となる、この対応は後述する。

## package.jsonなど

`next dev [dir]`で起動するソースディレクトリを指定できるので、これを利用する

```json
    "dev": "next dev app-mainapp",
    "dev:subapp": "next dev app-subapp -p 3002",
```

起動するとそれぞれのディレクトリの下に`.next`が出てくるので、`.gitignore` は下記のように追加

```
app-*/.next
```

また、同じくディレクトリごとに`tsconfig.json`が生成される。これも面倒なので下記のように`extends`を利用すると共通化しやすい

```json
{
  "extends": "../tsconfig.json",
}
```

## next.config.jsでwebpackの設定を変個する（共有ディレクトリがある場合）

デフォルトのnext.config.jsの場合、`app-xxx`の下のディレクトリしかコンパイルされなかったりモジュール解決されなかったりするので、共通ディレクトリがある場合は下記のようにwbpackの設定を入れると解決される。

```js
// next.config.js

const appendRootDir = (rule) => {
  if (!Array.isArray(rule?.include)) {
    return rule
  }
  // include設定が存在する場合に、コンパイル対象にするディレクトリを追加する
  rule.include = [...rule.include, __dirname]    
  return rule
}

module.exports = {
  webpack: (config) => {
    config.module.rules.map(rule => {
      if (Array.isArray(rule.oneOf)) {
        return {
          oneOf: rule.oneOf.map(rule => appendRootDir(rule))
        }
      }
      // next-swc-loaderのみ対象とする
      if (rule?.use?.loader !== "next-swc-loader") {
        return rule
      }
      return appendRootDir(rule)
    })
    return config
  }
}
```

next 12以上の場合、通常TypeScriptは`next-swc-loader`によって解決されるので、`next-swc-loader`のloaderに対して`include`設定を書き換える。

`__dirname`全部を対象としてしまうのがやりすぎな場合であれば、`__dirname/shared`などディレクトリを絞るのも良いだろう
