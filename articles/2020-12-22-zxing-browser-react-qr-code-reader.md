---
title: '@zxing/browserとReactの組み合わせでQR Code Reader作る'
emoji: 📸
type: tech
topics:
  - react
  - typescript
  - qrcode
  - zxing
published: true
---

QR Code Readerの実装を提供しているzxingだが、`@zxing/borwser`とパッケージが分離したりして微妙に色々変わってたので、Reactの組み合わせ事例をまとめてみる。

## Demo
※ カメラが開きます。ご注意ください
https://codesandbox.io/s/immutable-wildflower-jdk0h

## 実装

### installation

ひとまず@zxingをインストール。なお、今回使うコードは[chakra-ui](https://chakra-ui.com/)等も利用している

```
$ yarn add @zxing/browser @zxing/library
```

### コード

まずいきなりコアな部分を作っていく

```tsx
import { BrowserQRCodeReader, IScannerControls } from "@zxing/browser"
import { Result } from '@zxing/library'

const QrCodeReader : FC<{onReadQRCode: (text: Result) => void}>= ({ onReadQRCode }) => {
  const controlsRef = useRef<IScannerControls|null>()
  const videoRef = useRef<HTMLVideoElement>(null)

  useEffect(() => {
    if (!videoRef.current) {
      return 
    }
    const codeReader = new BrowserQRCodeReader()
    codeReader.decodeFromVideoDevice(
      undefined, 
      videoRef.current, 
      (result, error, controls) => {
        if (error) {
          return
        }
        if (result) {
          onReadQRCode(result)
        }
        controlsRef.current = controls
      })
    return () => {
      if (!controlsRef.current) {
        return
      }
      
      controlsRef.current.stop()
      controlsRef.current = null
    }
  }, [onReadQRCode])

  return <video
    style={{ maxWidth: "100%", maxHeight: "100%",height:"100%" }}
    ref={videoRef}
  /> 
 
}

```

まず`new BrowserQRCodeReader`でリーダーを初期化し、`decodeFromVideoDevice`を利用する。少し前のバージョンでは`decodeFromInputVideoDeviceContinuously`として提供されていた関数になり、カメラ情報から連続で読み続けてくれるものになる。
Worker等を使わずにお手軽に非同期連続読み取りしてくれるのは手軽で嬉しい。

`decodeFromVideoDevice(undefined)`と第一引数をundefinedにしている。
第一引数はもともとdeviceIdを指定するのだが、ここをundefinedにすると[`facingMode: "environment"`として、背面カメラ優先でカメラを利用してくれるらしい](https://github.com/zxing-js/library/blob/949f05c95a82387d0cb3e44470baeb21523683ad/src/browser/BrowserCodeReader.ts#L315-L317)。
第二引数には`<video>`要素に仕掛けるDOMのrefを指定している。

第三引数が結果受け取りのコールバックとなっており、ここで受け取った結果を利用している。
最後にクリーンナップ関数で利用したいため、`controls`の値をこれもrefsとして格納している。


あと残りは表示部分。特に珍しい部分も無いので、掲載のみとしている

```tsx

const QrCodeResult: FC<{qrCodes: string[] }> = ({qrCodes}) => {
  return <Table>
    <Tbody>
      {qrCodes.map((qr,i) => <Tr key={i}>
        <Td>
          <Fade in={true}>{qr}</Fade>
        </Td>
      </Tr>)}
    </Tbody>
  </Table>
}

const QrApp = () => {
  const [qrCodes, setQrCodes] = useState<string[]>([])

  return <ChakraProvider>
    <Container>
      <Flex flexDirection="column">
        <Box flex={1} height={"60vh"}>
          <QrCodeReader onReadQRCode={(result) => {
            setQrCodes((codes) => {
              return [result.getText(), ...codes]
            })
          }}/>
        </Box>
        <Box flex={1} height={"40vh"}>
          <Heading>Result</Heading>
          <QrCodeResult qrCodes={qrCodes}/>
        </Box>
      </Flex>
    </Container>
  </ChakraProvider>
}
```

## 他のQR実装

* [react-qr-reader](https://www.npmjs.com/package/react-qr-reader)
  * jsqrベース。workerに処理を流している
* [expo-camera(https://docs.expo.io/versions/latest/sdk/camera/)
  * Expo Webにしてしまえるならこれは便利。内部はjsqr実装
  * https://github.com/expo/expo/blob/4ae9f373059baabac079218f6fa0b5c5c929a152/packages/expo-camera/src/useWebQRScanner.ts#L39-L44