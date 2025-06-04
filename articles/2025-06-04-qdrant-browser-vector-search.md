---
title: ブラウザでvector化してQdrantで類似画像検索をやる
emoji: 🏹
type: tech
topics:
  - qdrant
  - typescript
  - tensor
published: true
---

ブラウザ上で画像をベクトル化し、[Qdrant](https://qdrant.tech/)を使って類似画像検索するやつ

## Qdrant準備

とりあえず準備としてDockerを使ってQdrantをローカル環境に構築。

```Dockerfile
version: '3.8'

services:
  qdrant:
    image: qdrant/qdrant:latest
    container_name: qdrant
    ports:
      - "6333:6333"
    volumes:
      - ./qdrant_data:/qdrant/storage
```

この設定でDockerコンテナを立ち上げると、ローカルの6333ポートでQdrantサーバーにアクセスできるようになる。

データ入れてから`http://localhost:6333/dashboard#/collections`のデータとか見ると楽しい。

## Vector化処理

次に今回のメインとなる画像をベクトル化する処理の部分。
Hugging Faceのtransformersライブラリをブラウザ対応版で利用し、CLIPモデルを使って画像をvector化すると良いっぽい

```ts
import { pipeline } from '@huggingface/transformers'
import type { ImagePipelineInputs, Tensor } from '@huggingface/transformers'

export const generateVector = async (source: ImagePipelineInputs) => {
  const extractor = await pipeline<"image-feature-extraction">('image-feature-extraction', 'Xenova/clip-vit-base-patch32')
  const queryVector: Tensor = await extractor(source)

  return queryVector.data
}
```

ブラウザで動くのか？と疑ったものの特に引っかかりもなく動いてくれた。すごい。

## collection作成

collectionはこんな感じで作成する。バッチで叩いてもいいし、アップロード前に実行する処理を挟むでもよい

```ts
export const targetCollectionName = 'dog'

export const qdrantClient = new QdrantClient({ host: "localhost", port: 6333 })

export const createCollection = async ( client = qdrantClient) => {
  const { exists } = await client.collectionExists(targetCollectionName)
  if (exists) {
    console.log(`Collection ${targetCollectionName} already exists.`)
    return
  }
  console.log(`Creating collection ${targetCollectionName}...`)
  await client.createCollection(targetCollectionName, {
    vectors: {
      size: 512, 
      distance: 'Cosine' 
    }
  })
}
```

## アップロードコンポーネント

画像をアップロードしてベクトル化し、Qdrantに保存するためのReactコンポーネントを作成する。

```tsx
import * as uuid from 'uuid'

const generateUuid = (url: string) => {
  return uuid.v5(url, uuid.v5.URL)
}

export const UploadPage: FC<UploadPageProps> = () => {
  const [files, setFiles] = useState<File[]>([])
  const [total, setTotal] = useState(0)
  const [processed, setProcessed] = useState(0)

  const uploadImage = async (file: File, description: string) => {
    // ローカルで画像をベクトル化
    const vector = await generateVector(file)
    const id = generateUuid(file.name)
    const qdrantClient = new QdrantClient({ host: "localhost", port: 6333 })
    // 画像を縮小してサムネイル用のbase64を作成。特に必須ではないが入れておくと便利
    const thumbnail = await fileToBase64(file, 300, 300)

    return await qdrantClient.upsert(targetCollectionName, {
      points: [
        {
          id,
          vector: Array.from(vector),
          payload: {
            description,
            filename: file.name,
            thumbnail
          }
        }
      ]
    })
  }

  const handleUpload = async () => {
    if (files.length === 0) return

    setTotal(files.length)
    setProcessed(0)

    for (const file of files) {
      await uploadImage(file, file.name)
      setProcessed(prev => prev + 1)
    }
  }

  const progressValue = total > 0 ? Math.round((processed / total) * 100) : 0

  return (
    <Stack gap={16}>
      <FileInput
        label="アップロードする画像ファイル"
        placeholder="画像ファイルを選択"
        accept="image/jpeg,image/png"
        multiple
        value={files}
        onChange={setFiles}
      />
      <Button
        onClick={handleUpload}
        disabled={files.length === 0}
      >
        アップロードを実行
      </Button>
      <Progress value={progressValue} />
    </Stack>
  )
}
```

Vector化してQdrantにupsertで更新している。
idは数値かuuidが利用できるが、同じファイルを一意化したかったのでuuid v5を利用している。
これも特にひっかかりなくブラウザから動いた。

## 検索ページコンポーネント

次に、画像を指定して類似画像を検索するためのコンポーネントを実装する。

```tsx
type VectorizePageProps = Record<string, never>

const SEARCH_LIMIT = 100

export const VectorizePage: FC<VectorizePageProps> = () => {
  const [selectedFile, setSelectedFile] = useState<File | null>(null)
  const [previewUrl, setPreviewUrl] = useState<string | null>(null)
  const [searchResults, setSearchResults] = useState<QdrantSearchResult | null>(null)
  const [isLoading, setIsLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const handleFileSelect = (file: File) => {
    // 画像ファイルのみを許可
    if (!file.type.startsWith('image/')) {
      setError('画像ファイルのみアップロード可能です')
      return
    }

    setSelectedFile(file)
    setError(null)

    const fileUrl = URL.createObjectURL(file)
    setPreviewUrl(fileUrl)

    setSearchResults(null)
  }

  const handleSearch = async () => {
    if (!selectedFile || !previewUrl) {
      setError('画像ファイルを選択してください')
      return
    }

    try {
      setIsLoading(true)
      setError(null)

      // 1. 画像をベクトル化
      const vectorData = await generateVector(previewUrl)
      const client = new QdrantClient({ host: "localhost", port: 6333 })

      // 2. 生成されたベクトルで検索実行
      const data = await client.search(targetCollectionName, {
        vector: Array.from(vectorData),
        limit: SEARCH_LIMIT,
      })

      setSearchResults(data)

    } catch (err) {
      setError(err instanceof Error ? err.message : 'ベクトル化または検索中にエラーが発生しました')
    } finally {
      setIsLoading(false)
    }
  }

  return (
    <Container size="md" py="xl">
      <Stack gap={16}>
        <Group justify="apart">
          <Title order={1}>画像検索</Title>
          <Button component="a" href="/insert" variant="outline">
            データ登録ページへ
          </Button>
        </Group>

        <ImageUploader
          onFileSelect={handleFileSelect}
          selectedFile={selectedFile}
          error={error}
        />

        {previewUrl && (
          <ImagePreview
            previewUrl={previewUrl}
            onSearch={handleSearch}
            isLoading={isLoading}
          />
        )}

        {searchResults && <SearchResultsList results={searchResults} />}
      </Stack>
    </Container>
  )
}
```

ユーザーが選択した画像をベクトル化して、そのベクトルを使って類似する画像をQdrantから検索している。

## 検索結果

実際に動かした結果はこんな感じ

![アップロード画面](https://storage.googleapis.com/zenn-user-upload/6aefb9520f08-20250604.png)

混ぜ込んだ同じ犬の結果が出たり（背景色に引っ張られてそう）、それ以外もわりかし近いものが出てる

![検索結果](https://storage.googleapis.com/zenn-user-upload/66c29bfc02e6-20250604.png)

別の例として、[CC0ライセンスの画像](https://www.flickr.com/photos/157635012@N07/44134376585)を使った検索でもそれっぽい感じになった

![別の検索例](https://storage.googleapis.com/zenn-user-upload/58062230c5fa-20250604.png)

たのしい