---
title: pillowでtextを泥臭くletter-spacingする
emoji: 🔠
type: tech
topics:
  - python
  - pillow
published: true
---

pillowでtextにletter spacingをかけたかったが、標準では用意されてなかったので自前した。

1文字ずつ`draw.text`する方法もありそうだったが、ベースラインを気にするとうまく行かない予感がしたので、描画 -> 切り出し -> paddingを計算して再結合 という手段をとった。

```python
def letter_spacing(font, text, letter_spacing):
    font_size = font.getsize(text)
    text_img = Image.new("L", font_size, color=255)
    
    draw = ImageDraw.Draw(text_img)
    draw.text((0, 0), text, font=font)

    # paddingを含めたサイズの計算・新しい画像の生成
    padded_width = font_size[0] + letter_spacing * (len(text) -1)
    padded_text_img = Image.new("L", (padded_width, font_size[1]), color=255)
    
    # 文字を切り出しつつ結合
    char_imgs = []
    cursor = 0
    for i, char in enumerate(list(text)):
        char_w = font.getsize(char)[0]
        end = cursor + char_w
        ws = (cursor + 1, end)
        char_img = text_img.crop((ws[0], 0, ws[1], text_img.height))
        
        new_paste_w = ws[0] + i * letter_spacing
        padded_text_img.paste(char_img, (new_paste_w, 0))
        cursor = end  # increment
    return padded_text_img

if __name__ == "__main__":
    
    img = Image.new("L", (600, 200), color=255)
    font = ImageFont.truetype("./Roboto-Bold.ttf", 40)
    text = "1,234 good morning"

    # paddingしてない版の描画
    font_size = font.getsize(text)
    text_img = Image.new("L", font_size, color=255)
    draw = ImageDraw.Draw(text_img)
    draw.text((0, 0), text, font=font)
    img.paste(text_img)

    # padding版の描画
    letter_spacing_img  = letter_spacing(font, text, 10)
    img.paste(letter_spacing_img, (0, font_size[1]))

    img.save("./output.png")
```

## 結果

![](https://storage.googleapis.com/zenn-user-upload/2fgmjnfehiprz72stkepfpeq6quy)
