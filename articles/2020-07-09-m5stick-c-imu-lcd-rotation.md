---
title: M5StickCが回転したら画面も回転するようにするやつ
emoji: 🌀
type: tech
topics:
  - m5stickc
  - arduino
published: true
terrier_export_from: "notion"
---

適当に放置するタイプだと回転してるときに自動回転させたい時があった。
`M5.IMU.getAhrsData`と`M5.Lcd.setRotation`で可能。

* https://github.com/m5stack/M5StickC/blob/b0872ddce2ae64fa19f6e62f4c19c7d89cb68b8b/src/IMU.h

```c
// 横
void display_rotation_horizontal()
{
  float pitch, roll, yaw;
  M5.IMU.getAhrsData(&pitch, &roll, &yaw);
  int rotation = (pitch < 0) ? 1 : 3;

  M5.Lcd.setRotation(rotation);
}

// 縦
void display_rotation_vertical()
{
  float pitch, roll, yaw;
  M5.IMU.getAhrsData(&pitch, &roll, &yaw);
  int rotation = (roll < 0) ? 2 : 4;

  M5.Lcd.setRotation(rotation);
}
```


::: details 全対応版も一応作ったものの、若干buggyかった


```c
int detect_rotation()
{
  float pitch, roll, yaw;
  M5.IMU.getAhrsData(&pitch, &roll, &yaw);
  int rotation = (abs(pitch) > abs(roll)) ? (pitch < 0) ? 1 : 3
                                  : (roll < 0) ? 2 : 4;
  M5.Lcd.setRotation(rotation);
}
```
:::

参考

