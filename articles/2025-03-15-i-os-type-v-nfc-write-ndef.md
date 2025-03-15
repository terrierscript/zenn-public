---
title: iOSでType-VのNFCへwriteNDEFで書き込む前にフォーマットが必要
emoji: 🐔
type: tech
topics:
  - swift
  - ios
  - nfc
published: true
---

Type-VのNFCタグにNDEFを書き込もうと`writeNDEF`を利用したらエラーが出た。

どうやらType-V(iso15693)の場合、書き込む前にCC（Capability Container）領域の初期化が必要で、このあたりはCoreNFCはやってくれないらしいので、フォーマット部分を自前した。

* https://stackoverflow.com/questions/61622309/base-address-requirement-for-ndef-messages-on-type-5-tags
* https://developer.apple.com/forums/thread/671513

探してもあまり正式な情報が見つからず、他のNFC書き込みアプリで書き込んだメモリ結果を読み取りながらギリギリ合わせたので、対応しないタグなどがあるかも。

# フォーマット処理の全体像

`formatISO15693TagToNDEF`関数でタグからシステム情報を取得し、CCブロックの構築とNDEFデータ領域の確保を行う。

```swift
func formatISO15693TagToNDEF(tag: NFCISO15693Tag, session: NFCTagReaderSession) async throws {
    let info = try await getSystemInfo(tag: tag)
    
    let blockSize = info.blockSize
    let ndefMagicNumber: [UInt8] = [0xE1, 0x40] // NFCフォーラム Type-V タグを示す
    let versionNumber: UInt8 = 0x01
    let ccBlock: [UInt8] = ndefMagicNumber + [0x30, versionNumber]
    
    let ndefTagFlag: UInt8 = 0x03  // NDEFメッセージTLVを示す
    let length: [UInt8] = encodeTLVLength(info.blockSize)
    let startTLV: [UInt8] = [ndefTagFlag] + length
    
    let header: [UInt8] = ccBlock + startTLV
    // メッセージを書き込めるように埋める
    var paddedMessage: [UInt8] = header
    let remainder = header.count % blockSize
    if remainder != 0 {
        paddedMessage += Array(repeating: 0, count: blockSize - remainder)
    }
    let blocks = splitMessageIntoBlocks(message: paddedMessage)
    try await writeBlocksSequentially(tag: tag, blocks: blocks)
}
```

## formatISO15693TagToNDEF関数

### 処理の概要

NDEFメッセージ書き込みの前処理としてType-V NFCタグを初期化する関数。処理は以下の5ステップで構成される。

1. タグからシステム情報を取得

```swift
let info = try await getSystemInfo(tag: tag)
```

ブロックサイズなど以降の処理に必要な情報がここに格納される。

2. CCブロック（Capability Container）の構築

```swift
let blockSize = info.blockSize
let ndefMagicNumber: [UInt8] = [0xE1, 0x40] // NFCフォーラム Type-V タグを示す
let versionNumber: UInt8 = 0x01
let ccBlock: [UInt8] = ndefMagicNumber + [0x30, versionNumber]
```

CCブロックの各バイトの役割:
* `[0xE1, 0x40]`: Type-Vタグのマジックナンバー
* `0x30`: 読み書き可能を示すアクセス権限バイト
* `0x01`: NDEFマッピングバージョン

3. TLV（Type-Length-Value）構造の構築

```swift
let ndefTagFlag: UInt8 = 0x03  // NDEFメッセージTLVを示す
let length: [UInt8] = encodeTLVLength(info.blockSize)
let startTLV: [UInt8] = [ndefTagFlag] + length
```

* Type: `0x03`（NDEFメッセージ識別子）
* Length: タグ容量から計算
* Value: 後続のNDEFメッセージ格納領域

4. ブロックサイズ調整のためのパディング処理

```swift
let header: [UInt8] = ccBlock + startTLV
var paddedMessage: [UInt8] = header
let remainder = header.count % blockSize
if remainder != 0 {
    paddedMessage += Array(repeating: 0, count: blockSize - remainder)
}
```

Type-Vタグはブロック単位での書き込みが必須のため、データをブロックサイズに調整する。

5. データの分割と書き込み

```swift
let blocks = splitMessageIntoBlocks(message: paddedMessage)
try await writeBlocksSequentially(tag: tag, blocks: blocks)
```

調整済みデータをブロックサイズで分割し、タグに順次書き込む。`writeBlocksSequentially`については後述
これにより`writeNDEF`が利用可能な状態となる。

## 各関数の実装

### システム情報の取得

コールバック地獄は避けたかったので、`withCheckedThrowingContinuation`で包んでasync/awaitにした。

```swift
func getSystemInfo(tag: NFCISO15693Tag) async throws -> NFCISO15693SystemInfo {
    try await withCheckedThrowingContinuation { continuation in
        tag.getSystemInfo(requestFlags: [.highDataRate]) { result in
            switch result {
            case .success(let systemInfo):
                continuation.resume(returning: systemInfo)
            case .failure(let error):
                continuation.resume(throwing: error)
            }
        }
    }
}
```

### メッセージのブロック分割

タグに書き込むデータはブロックサイズに合わせて分割する必要があるのでその処理

```swift
private func splitMessageIntoBlocks(message: [UInt8], blockSize: Int = 4) -> [Data] {
    var blocks: [Data] = []
    for i in stride(from: 0, to: message.count, by: blockSize) {
        let block = Array(message[i..<i + blockSize])
        blocks.append(Data(block))
    }
    return blocks
}
```

### ブロックの書き込み

分割したブロックを順番にタグに書き込んでいく。
`tag.writeSingleBlock`で直接書き込みの力業が必要

```swift
private func writeBlocksSequentially(tag: NFCISO15693Tag, blocks: [Data]) async throws {
    for (index, block) in blocks.enumerated() {
        let blockNumber = UInt8(index)
        try await tag.writeSingleBlock(
            requestFlags: [.highDataRate], blockNumber: blockNumber, dataBlock: block)
    }
}
```

### TLV長のエンコード処理

255バイト以上だと3バイト構造に変える必要があったので、その部分を実装

```swift
func encodeTLVLength(_ length: Int) -> [UInt8] {
    if length < 255 {
        // 1バイト形式: 0x00 〜 0xFE
        return [UInt8(length)]
    } else {
        // 3バイト形式: 0xFF + 2バイト（ビッグエンディアン）
        let highByte = UInt8((length >> 8) & 0xFF)  // 上位バイト
        let lowByte = UInt8(length & 0xFF)  // 下位バイト
        return [0xFF, highByte, lowByte]
    }
}
```

# 使用例

NFCセッションを作って実際に書き込む

```swift
class NFCWriter: NSObject, NFCTagReaderSessionDelegate {
  // ...
    func tagReaderSession(_ session: NFCTagReaderSession, didDetect tags: [NFCTag]) {
        guard let firstTag = tags.first else { return }
        
        switch firstTag {
          // ...
        case .iso15693(let tag):
            Task {
                do {
                    try await session.connect(to: firstTag)
                    try await formatISO15693TagToNDEF(tag: tag, session: session)
                    
                    let payload = NFCNDEFPayload.wellKnownTypeTextPayload(
                        string: "Hello",
                        locale: Locale(identifier: "en")
                    )!
                    let message = NFCNDEFMessage(records: [payload])
                    
                    let status = try await tag.queryNDEFStatus()
                    try await tag.writeNDEF(message)
                    
                    self.message = "「Hello」を書き込みました"
                    session.invalidate()
                } catch {
                    self.message = "エラー: \(error.localizedDescription)"
                    session.invalidate()
                }
            }
        }
    }
}
```

`writeNDEF`の前に`formatISO15693TagToNDEF`を呼び出すのがキモ。これでNDEFメッセージが書き込める。
