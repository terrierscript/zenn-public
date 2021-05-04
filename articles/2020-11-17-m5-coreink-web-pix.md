---
title: M5Stack CoreInkをWeb経由で取得して表示するやつ作った記録
emoji: ✒️
type: idea
topics:
  - arduino
  - nextjs
  - m5stack
published: true
---

↓これやったやつのメモ。技術的な正確性よりもログに近い記事のためtechではなくidea。
@[tweet](https://twitter.com/terrierscript/status/1328363600117829632)

## モチベーション
[M5Stack CoreInk](https://www.switch-science.com/catalog/6735/)を買ってみた。
特徴としてはE-inkを内蔵していることだが、[shutdown](http://docs.m5stack.com/#/en/api/coreink/system_api?id=shutdown)を呼び出すことで定期起動させることで長時間フォトフレーム的なことが出来たら面白そうに感じた。

しかしarduinoでコードを書いて色々やるのはなかなかしんどい。そこでWeb経由に出来ないかと色々やりたかった

## 前準備

* この辺を見てarduinoをセットアップする
  * https://qiita.com/youtoy/items/507ecc93e71e60b7f5ce
  * https://docs.m5stack.com/#/en/arduino/arduino_development 
* M5-CoreInkのライブラリは残念ながら現在の0.0.1では上記のshutdownがまだなかったので、masterをzipでダウンロードする必要があった
  * https://github.com/m5stack/M5-CoreInk
  * 「スケッチ」ｰ>「ライブラリをインクルード」-> 「ZIP形式のライブラリをインストール」
  * see: https://www.mgo-tec.com/arduino-ide-lib-zip-install

## 実装

最終的にこんな具合の方針になった
* なるべくCoreink側は表示するだけに徹させる
* 色々やってみた結果、JSON形式で受け取ってArduinoJsonでパースしようとするとメモリリークして無理っぽいので、
* ソース: https://github.com/terrierscript/arduino-coreink-WebPix

### サーバー側(next.js)

サーバー側はnext.jsで行った。jimpでデータを生成する。

```js
import Jimp from 'jimp';

export const render = () : Promise<Jimp> => {
  return new Promise((resolve, reject) => {
    new Jimp(200, 200, '#ffffff', async (err, image) => {
      image.scan(100, 100, 50, 50, function (x, y, offset) {
        this.bitmap.data.writeUInt32BE("#ffffff", offset, );
      });
      resolve(image)
    })
  })
}
```

この画像データをこんな具合にCoreInk用に変換する。

1. bitmapを二値化
2. 8桁ずつまとめて、16進数2桁にする
3. joinする

```js
import split from "just-split"
export const imageToPixhex = (image) => {
  const data = image.bitmap.data
  let binmap = []
  image.scan(0, 0, image.bitmap.width, image.bitmap.height, (x, y, idx) => {
    // console.log(x,y,data[idx + 0])    
    const bb = (data[idx + 0] + data[idx + 1] + data[idx + 2]) / 3
    const bin = (bb < 128) ? 0: 1
    binmap.push( bin)
    return bin
  })

  const chunks = split(binmap, 8)
  const pixhex = chunks.map(c => {
    return parseInt(c.join(""),2).toString(16).padStart(2, "0")
  })
  return pixhex.join("")
}
```

あとはこれをstringとして表示する

```js
export default async (req, res) => {
  const image = await render();
  const pixhex = imageToPixhex(image)
  res.json(pixhex.join(""))
  res.end()
}
```

### クライアント(CoreInk)側

[だいたいこんな具合](https://github.com/terrierscript/arduino-coreink-WebPix/blob/aab069fb68365cc65ac898c2b52259ccc4d3fff3/arduino/WebPix.ino#L18-L57)

まずこんな具合でデータ取得。自前のデータを利用するのでエラーハンドリングとかは雑になってる

```cpp
String loadPage()
{
  HTTPClient http;
  http.begin(endpoint);
  int httpCode = http.GET();
  String result = "";
  if (httpCode > 0)
  {
    result = http.getString();
  }
  else
  {
    Serial.println("Error on HTTP request");
  }
  http.end();
  return result;
}
```

あとは取得したデータを`unsigned char`に戻していく。
また、Spriteはclearしない場合白が上書きされないような実装になっているようだったので（[たぶん](https://github.com/m5stack/M5-CoreInk/blob/b1a499954b5e98cfbca74f32ff9d17b4c6cdba35/src/utility/Ink_Sprite.cpp#L87-L89)）、`M5.M5Ink.drawBuff`を直接使うことにした

```cpp
int SIZE = 5000;
unsigned char pool[5000];
void loadData()
{
  // 1が白、0が黒。全部白で初期化
  for (int i = 0; i < SIZE; i++)
  {
    pool[i] = 0xff;
  }
  // データ取得
  String buf = loadPage();
  for (int i = 0; i < SIZE; i++)
  {
    // substringで2文字ずつ取得
    String pp = buf.substring(i * 2, i * 2 + 2);
    // intに戻す
    int bbb = (int)strtol(pp.c_str(), 0, 16);
    pool[i] = bbb;
  }

  delay(3000);
  // 描画
  M5.M5Ink.drawBuff((uint8_t *)pool);
}
```

あとセットアップで呼び出す。
今回は`loop`を使わずすべて`setup`にまとめてしまっている

```cpp
void setup()
{
  M5.begin();
  if (!M5.M5Ink.isInit())
  {
    Serial.printf("Ink Init faild");
    while (1)
      delay(100);
  }
  
  connectWifi(); // wifi接続
  delay(1000);
  
  Serial.printf("serial\n");

  if (M5.BtnMID.isPressed()) // 真ん中ボタンを押したら終了
  {
    Serial.printf("Btn %d was pressed \r\n", BUTTON_EXT_PIN);
    digitalWrite(LED_EXT_PIN, LOW);
    M5.M5Ink.clear();
    M5.shutdown();
  } else { // 通常は描画
    loadData();
    digitalWrite(LED_EXT_PIN, LOW);
    M5.shutdown(120); // 適当に秒数をおいて再起動
  }

}
```

wifi接続処理は[こちら](https://github.com/SWITCHSCIENCE/samplecodes/blob/0a395013a71ee79e01936e1416b40f8951327e75/M5CoreInk/m5coreink_ntp_rtc/m5coreink_ntp_rtc.ino#L178-L186)を参考にした


書き込みはarduino-cli使った

```sh
$ arduino-cli compile -b m5stack:esp32:m5stack-coreink WebKal.ino -u -p /dev/cu.SLAB_USBtoUART
```


## 拡張：犬を表示する

あとはサーバー側で表示するものを変える。犬を出そう。

@[tweet](https://twitter.com/terrierscript/status/1328658817412808710)

今回は[Dog API](https://dog.ceo/dog-api/)を利用することにした。

```js

const cropSize = (image) => {
  const {width , height} =image.bitmap
  return {
    x: Math.max(width / 2 - 200, 0),
    y: Math.max(height / 2 - 200, 0),
  }
}
export const render = async (): Promise<Jimp> => {
  const { data } = await axios.get("https://dog.ceo/api/breeds/image/random");
  const imgUrl = data.message
  return new Promise((resolve, reject) => {
    Jimp.read(imgUrl)
      .then(image => {
        const {width , height} =image.bitmap
        // ざっくりresize
        const resize = width < height ? [200, Jimp.AUTO] : [Jimp.AUTO, 200]
        image.resize(resize[0], resize[1])
        
        // M5-CoreInk自体を横向きにしたいので270度回転
        image.rotate(270)

        // ざっくりcrop
        const crop = cropSize(image)

        // グレースケール化して
        image.crop(crop.x, crop.y, 200,200)
          .grayscale()
        image.bitmap = floydSteinberg(image.bitmap)
      
        // image.resize(200,200)
        resolve(image)
      })
  })
}
```

グレースケール化周りは[jimpのissue](https://github.com/oliver-moran/jimp/issues/212#issuecomment-383429606)に解決策が掲示されていた。


## その他学び・知見
* `Guru Meditation Error: Core  1 panic’ed (Unhandled debug exception)`が不定期に起きて困ったが、どうやらメモリリークっぽかった。
* 躓いたときは、Sketchを小さく作って試すとやりやすい（JSONを試したりするのは小さく作ったほうが良かった）
* arduino-cliはシリアルモニタの機能を持ってないので、`screen`で代用する必要があった
  * `$ screen /dev/cu.SLAB_USBtoUART 115200`
  * screenを閉じても接続はつながっているので、消すときは`$ pkill SCREEEN`


## 参考文献
* https://github.com/wararyo/coreink-weather
* https://github.com/ksasao/ImageToCoreInk