---
title: react-three-fiberを使ってJSXを3Dプリントする
emoji: ♟
type: tech
topics:
  - react
  - reactthreefiber
  - threejs
  - stl
  - 3dprint
published: true
terrier_date: '2019-12-29T14:03:47.201Z'
terrier_export_from: 'https://blog.terrier.dev/blog/2019/20191229230347-3d-printing-react-component'
---
> 記事作成日:2019-12-29

[react-three-fiber](https://github.com/react-spring/react-three-fiber)と[three.jsのExporter](https://github.com/mrdoob/three.js/blob/dev/examples/jsm/exporters/)を使ってJSXを3Dプリントする事に成功した

![](https://storage.googleapis.com/zenn-user-upload/2ebszuoer89jhnyz8lrxrd162a33)

今回はそのPoCについて記述する

## どういうことか？

react-three-fiberはThree.jsをReact上で取り扱うためのライブラリになる。

このライブラリは特性上、`mesh`や`geometry`といったポリゴンを構成する部分をJSXとして記載できる。

```jsx
function Thing() {
  return (
    <mesh>
      <boxBufferGeometry attach="geometry" args={[1, 1, 1]} />
      <meshNormalMaterial attach="material" />
    </mesh>
  )
}
```

そして、3Dプリンタにかけるにはthree.jsのデータをSTLやOBJ,glTFあたりに変換する必要がある。
これらもthree.jsにSTLExporterやGLTFExporterなど各種フォーマットが取り揃えられている（一部Exporterはundocumentedなので不安定な可能性はある）

これらを組み合わせることで「JSXを3D印刷する」ということが叶いそうだ。

## Demo

https://codesandbox.io/s/react-three-fiber-to-stl-or-gltf-v5-lsvn2?fontsize=14&hidenavigation=1&module=%2Fsrc%2Fmodel%2FModel.tsx&theme=dark

Model.tsxを編集するとSTLやglTFが出力される（ちょっと重いので注意）

## 細かい部分

### Modelの生成

Modelは今回このような感じで生成した。`BufferGeometry`でも`Geometry`でも使えるし、ネストしても問題なさそうだった。

```tsx
// Model.tsx
import { Canvas, useFrame, useThree } from "react-three-fiber"
import React from "react"

export const Model = () => {
  return (
    <mesh>
      <Model1 />
      <Model2 />
    </mesh>
  )
}
const Model1 = () => {
  return (
    <mesh position={[0, 0, 0]}>
      <cylinderBufferGeometry attach="geometry" args={[5, 5, 5]} />
      <meshNormalMaterial attach="material" />
    </mesh>
  )
}
const Model2 = () => {
  return (
    <mesh>
      <mesh position={[-5, -1.5, -3]}>
        <boxBufferGeometry attach="geometry" args={[6, 2, 5]} />
        <meshNormalMaterial attach="material" />
      </mesh>
      <mesh>
        <mesh position={[0, 3, -1]}>
          <octahedronBufferGeometry attach="geometry" args={[4]} />
          <meshNormalMaterial attach="material" />
        </mesh>
        <mesh position={[3, 0.5, 3]}>
          <sphereGeometry attach="geometry" args={[3, 10, 32]} />
          <meshNormalMaterial attach="material" />
        </mesh>
      </mesh>
    </mesh>
  )
}
```

あとはこんな感じにすればモデルを描画できる

```tsx
const App = () => {
  const ref = useRef()
  const { gl } = useThree()
  gl.setClearColor("#ff99cc")

  return <Canvas>
    <Model />
  </Canvas>
}
```

### sceneをtraverseして、exporterに対応できるようにする

STLへの変換自体は、先に言った通り`STLExporter`を使えば一応できる。
`useThree`で取得できる`scene`をこれに渡せばよい

```jsx
// ExportStl.tsx
import { STLExporter } from "three/examples/jsm/exporters/STLExporter"

export const ExportStl = () => {
  const { scene } = useThree()
  useEffect(() => {
    const stl = new STLExporter().parse(scene)
    console.log(stl)
  }, [scene])
  return <mesh></mesh>
}
```

これを先程と同じくCanvas下に配置すれば良い

```tsx
const App = () => {
　// ...
  return <Canvas>
    <Model />
    <ExportStl />
  </Canvas>
}
```



ただ、利用するgeometryなどでエラーが起きやすかったり、geometry同士をmergeしないことでバラバラなオブジェクトとして吐き出されるのが印刷データを作るのに都合が悪かったため、事前に下記のように変換することにした。

```js
import {
  Mesh,
  MeshBasicMaterial,
  Scene,
  Geometry,
  BufferGeometry,
  Object3D
} from "three"

export const toRenderble = (scene: Scene): Scene => {
  let tmpGeometry = new Geometry()

  scene.clone().traverse((mesh) => {
    if (!isMesh(mesh)) return
    if (!mesh.geometry) {
      return
    }
    // 利用できるBufferGeometryをGemometryに揃える（後述）
    const appendGeom = toRenderableGeometry(mesh.geometry)
    if (!appendGeom) {
      return null
    }

    // 親のmeshを考慮する
    if (mesh.parent) {
      mesh.parent.updateMatrixWorld()
      mesh.applyMatrix(mesh.parent.matrixWorld)
    }

    mesh.geometry = appendGeom
    // meshをマージしていく
    tmpGeometry.mergeMesh(mesh)
  })

  // 最後、更にそれをBufferGeometryに変換する
  const outputScene = new Scene()
  const buf = new BufferGeometry().fromGeometry(tmpGeometry)
  const mesh = new Mesh(buf, new MeshBasicMaterial())
  outputScene.add(mesh)
  return outputScene
}
```

BufferGeometryをGeometryに一度変換しているところでは、一部変換できないBufferGeometryで失敗することがあった（react-three-fiberが挿入しているのかも？）ので、それらはスキップしている

```js
const toRenderableGeometry = (
  geom: Geometry | BufferGeometry
): Geometry | null => {
  if (isGeometry(geom)) {
    return geom
  }
  if (geom.index === null && !geom.getAttribute("position")) {
    return null
  }

  // わりとfromBufferGeometryに失敗するパターンはありそうなので、失敗したらnullとしてしまう
  try {
    const buf = new Geometry().fromBufferGeometry(geom)
    return buf
  } catch (e) {
    console.warn(`skip: ${geom}`)
    return null
  }
}

```

あとはこれを先程のComponentで呼び出せば良い。結果は今回はContextに渡す事にする（この点については後述）

```jsx
export const ExportStl = () => {
  const { scene } = useThree()
  const { setStl } = useExporterStore()
  useEffect(() => {
    const copyScene = toRenderble(scene)
    const stl = new STLExporter().parse(copyScene)
    setStl(stl)
  }, [scene])
  return <mesh></mesh>
}
```

下記のようにhooks化するのも良いだろう

```js
export const useSTLExporter = () => {
  const { scene } = useThree()
  const [result, setResult] = useState()
  useEffect(() => {
    const copyScene = toRenderble(scene)
    const stl = new STLExporter().parse(copyScene)
    setResult(stl)
  }, [scene])
  return result
}
```

glTFの場合は下記のようになる

```tsx
const exportGltf = (scene, cb) => {
  return new GLTFExporter().parse(
    scene,
    (obj) => {
      cb(JSON.stringify(obj, null, 2))
    },
    { trs: true }
  )
}

export const ExportGltf = () => {
  const { scene } = useThree()
  useEffect(() => {
    const copyScene = toRenderble(scene)
    exportGltf(copyScene, (glTF) => {
      console.log(glTF)
    })
  }, [scene])
  return <mesh></mesh>
}
```

### modelの変換結果を外に渡せるようにする
上記でContextを利用する話を記載したが、react-three-fiberはreconcilerを使ってカスタムレンダラーになっているため`<Canvas>`より子のコンポーネントで通常のように`useContex`などが簡単に使えない。

そこで[このissue](https://github.com/react-spring/react-three-fiber/issues/262)を参考に下記のように中間の受け渡し口を作った。

```jsx

// App.tsx

const App = () => {
  return (
    <div>
      <ExporterStoreProvider> {/* ここはそのまま利用する */}
        <World />
      </ExporterStoreProvider>
    </div>
  )
}

```

```jsx
// World.tsx

export const World = () => {
  const value = useExporterStore() // 一度値を取り出す
  return (
    <Canvas camera={{ position: [0, 0, 30] }}> 
      <ExportPassProvider value={value}> {/* Canvas内部で再度Providerにわたすことで中継させる */}
        <Model />
        <ExportStl />
      </ExportPassProvider>
    </Canvas>
  )
}

```

## プリントまでの流れ

流石にSTLデータ以降はReactでは出来ないので、[Ultimaker cura](https://ultimaker.com/software)などでgcodeに変換した。

![](https://storage.googleapis.com/zenn-user-upload/zc1geqhi49sufkim3oodw3b0nepv)

そしてこれを3Dプリントすれば完了だ
（あまりきれいにプリント出来てないのはご容赦いただきたい）

![](https://storage.googleapis.com/zenn-user-upload/vyhve3tisbu4e5btk9902b1bockq)

## まとめ

パフォーマンスの難点やまだ考慮されてないパターンなどはあるが、ひとまず目的は達成できた。JSXでの構築は[この記事のカバー画像に使ったReactのロゴ](https://github.com/terrierscript/poc-react-three-stl-exporter/blob/master/src/model/ModelReactLogoCoin.tsx#L6-L101)のような規則性があるようなものは非常に作りやすい。これらをパーツとして、他のツールで全体を組み立てることを考えれば有益な可能性を感じている
