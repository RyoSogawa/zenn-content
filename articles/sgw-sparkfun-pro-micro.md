---
title: "SparkFun Pro Microを使って自作キーボードをUSB-C化した記録"
emoji: "⌨️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["keyboard"]
published: true
---

USB-C 対応の SparkFun Pro Micro を購入しセットアップするときに躓いたので、そのときの手順を記録します。

## 前置き（飛ばしてください）
自作キーボードの世界と出会って早 1 年。
割れキーと自由なレイアウトはタイピングをとても快適にしてくれました。

そんな中、不満点が出てきたのは持ち運びのこと。
割れキーはばらしたり組み立てたり、ケーブルが必要だったり、ラップトップの上に起きにくかったりしてあまり持ち運びに向いてないことが分かってきました。

そんな中先日素晴らしいキーボードと出会い、外用として 2 台目の自作キーボードを作成しました。

https://twitter.com/sgw123456/status/1456984807389364224?s=20

`cocot49` というこのキーボードは。

- 非分割
- 40%
- カラムスタッガード配列
- 親指キー4 つ
- 左右で腕が多少開くように逆ハの字型になっている

という、自分が外用キーボードに求めるすべての要素を兼ね揃え、
それだけでなく、

- トラックボール、ロータリーエンコーダ搭載
- アクリル積層プレートとシリコンシートのオプションあり

という特徴も持ち合わせたオールインワンキーボードでした。

使用感の満足度がとても高かったこともあってか、逆に不満点が浮き彫りになってきました。

## microUSBとUSB-Cの変換ケーブルは種類が少ない
これに尽きます。

自分は以下の条件を満たすケーブルが欲しかったのですが、探しても探しても見つかる気配がありませんでした。

- USB-C 〜 microUSB の端子になっている（Macbook に直接接続できる）
- Macbook 側は通常の L 字端子
- microUSB 端子はマグネットの L 字端子

変換端子等も検討しましたがやはりスマートでない。。

それに対して USB-C 〜 USB-C のケーブルだと。

- L 字 〜 ストレートのケーブル
- ストレート → L 字変換のマグネット端子

を購入するだけでそれが実現できそうではないか！

※実際に購入したものはこちら。

https://www.amazon.co.jp/dp/B09JKZDN2G/ref=cm_sw_em_r_mt_dp_QW12G031D8GRJ0AGAQM7

https://www.amazon.co.jp/dp/B08LMGQ8KD/ref=cm_sw_em_r_mt_dp_HQ465ND8CW480EYP919P?_encoding=UTF8&psc=1


ということで ProMicro を USB-C 対応したものに変更することにしました。


## まずは買い物
遊舎工房さんで購入。

https://shop.yushakobo.jp/products/sparkfun-qwiic-pro-micro-usb-c-atmega32u4?variant=37665305460897

※後々気づきましたが別でもう少し安く買えるかも？

:::message alert
コンスルーも同時に通常サイズのものを買いましたが、USB-C は microUSB よりも高さがある関係でサイズがちょっと合いませんでした。
3.5mm のコンスルーがよさそうです。
:::
https://shop.dailycraft.jp/products/conthrough_12_35

## セットアップ手順
[公式ドキュメント](https://learn.sparkfun.com/tutorials/qwiic-pro-micro-usb-c-atmega32u4-hookup-guide?_ga=2.191154986.1844118759.1581833895-706105357.1557762198#example-3-qwiic-enabled-hid-mouse-and-keyboard)

Chrome に日本語訳してもらいながら読みました。
だいたい飛ばし飛ばし読んでます。

### 1. ドライバーインストール
https://learn.sparkfun.com/tutorials/qwiic-pro-micro-usb-c-atmega32u4-hookup-guide?_ga=2.191154986.1844118759.1581833895-706105357.1557762198#installing-drivers-mac-and-linux

手順が書かれてますが同じようにセットアップアシスタントの画面は開きませんでした。。
問題ないのかも？

Windows の方は 1 つ上の見出しをどうぞ。

### 2. Arduino
https://learn.sparkfun.com/tutorials/qwiic-pro-micro-usb-c-atmega32u4-hookup-guide?_ga=2.191154986.1844118759.1581833895-706105357.1557762198#setting-up-arduino

こちらに書かれている通りですが、Arduino というソフトをインストールして起動します。

次に「設定」を開き、`追加のボードマネージャのURL` の欄に。
```
https://raw.githubusercontent.com/sparkfun/Arduino_Boards/master/IDE_Board_Manager/package_sparkfun_index.json
```
を入力し、「OK」でウインドウを閉じます。

![img](https://storage.googleapis.com/zenn-user-upload/f6ea7ac28bfa-20211203.png)

`「ツール」 > 「ボード」 > 「ボードマネージャ」` を選択。

検索欄に `sparkfun` と入力すると `SparkFun AVR Boards` という項目が出てくるので「インストール」をクリック。
※↓画像はインストール済みの状態です。

![img](https://storage.googleapis.com/zenn-user-upload/63c407ce3ac6-20211203.png)

「ツール」から、
- 「ボード」= `「SparkFun AVR Board」 > 「SparkFun Pro Micro」`
- 「プロセッサ」=`5V` のほう
- 「ポート」= `/dev/su.usbmodemXXXXXX` みたいなやつ（COM ポート?）

を選択。

動作確認例として挙げられている以下コードを Arduino の初期画面にコピペしてから「マイコンボードに書き込む」のボタンを押すとしばらくすると LED が交互に点灯するようになります。
※このステップは飛ばせるかも。

```
/* Pro Micro Test Code
   by: Nathan Seidle
   modified by: Jim Lindblom
   SparkFun Electronics
   date: September 16, 2013
   license: Public Domain - please use this code however you'd like.
   It's provided as a learning tool.

   This code is provided to show how to control the SparkFun
   ProMicro's TX and RX LEDs within a sketch. It also serves
   to explain the difference between Serial.print() and
   Serial1.print().
*/

int RXLED = 17;  // The RX LED has a defined Arduino pin
// Note: The TX LED was not so lucky, we'll need to use pre-defined
// macros (TXLED1, TXLED0) to control that.
// (We could use the same macros for the RX LED too -- RXLED1,
//  and RXLED0.)

void setup()
{
  pinMode(RXLED, OUTPUT);  // Set RX LED as an output
  // TX LED is set as an output behind the scenes

  Serial.begin(9600); //This pipes to the serial monitor
  Serial.println("Initialize Serial Monitor");

  Serial1.begin(9600); //This is the UART, pipes to sensors attached to board
  Serial1.println("Initialize Serial Hardware UART Pins");
}

void loop()
{
  Serial.println("Hello world!");  // Print "Hello World" to the Serial Monitor
  Serial1.println("Hello! Can anybody hear me?");  // Print "Hello!" over hardware UART

  digitalWrite(RXLED, LOW);   // set the RX LED ON
  TXLED0; //TX LED is not tied to a normally controlled pin so a macro is needed, turn LED OFF
  delay(1000);              // wait for a second

  digitalWrite(RXLED, HIGH);    // set the RX LED OFF
  TXLED1; //TX LED macro to turn LED ON
  delay(1000);              // wait for a second
}
```


### 3. QMK Toolbox
ファームウェアを書き込む前にコンスルーでキーボードと接続します。
SparkFun Pro Micro はコンスルー対応（？）していて、Pro Micro 側にもはんだ付けが不要です。
上下方向やコンスルーの接続向きは通常の Pro Micro と同様です。

次にファームウェアを書き込むために hex ファイルを用意します。

※cocot49 は[ビルドガイド](https://github.com/aki27kbd/cocot46/blob/main/doc/buildguide.md#%E3%83%95%E3%82%A1%E3%83%BC%E3%83%A0%E3%82%A6%E3%82%A7%E3%82%A2)から。

1. QMK Toolbox の `Local file` の欄で同ファイルを選択
2. `Auto-Flash` にチェックを入れて
3. 本体のリセットスイッチを**ダブルクリック**

:::message
通常の ProMicro と同様にリセットスイッチを普通にクリックすると、ダブルクリックしてくれという旨のエラーメッセージが表示され失敗します。
:::

### 4. あとはリマップ等
ここまでですでにキーボードとして動作するようになりました！
あとは REMAP 等を使ってレイアウトを設定すれば完了です！

https://remap-keys.app/

お疲れ様でした！🍵

:::message alert
ちなみにcocot49ではこのProMicroを使うとトラックボールが反応しなくなってしまうようです。  
くれぐれも自己責任でお願いします！
:::


## 余談
### REMAPでのレイヤー切り替えキー
REMAP を初めて使ったのですが、デフォルトで設定されているレイヤーキー用の、
- `無変換 Layer(1)`
- `変換 Layer(2)`

の設定はキーの Tap と Hold を検出している関係でか、Tap の設定を切っても Layer キーとして判定されるまでにちょっと時間がかかってストレスがありました。

`HOLD/TAP` ではなく、通常の `KEY` で Keycode=`TT(1)` 等に設定することで瞬時にレイヤーキーとして動作させることができます。

（[キーマップ](https://remap-keys.app/catalog/rZnKrpVyZLRcMqKgEM7Z/keymap?id=Qp0tdcxdcqjYE9T35Qbd)を公開しました）

※英/かな切り替えは、Karabiner を使って左右 Cmd キーの Tap に割り振っています。

### Macbookのキーボードをオフにする
もともと外用にこのキーボードを購入したのですが、Macbook 上に乗っける尊師スタイルではキーボードが押されてしまうサイズ感でした。

これも Karabiner の device 設定で cocot49 を接続したら Macbook のキーボードがオフになるように設定することで自動的に切り替わるように設定できました。

![](https://storage.googleapis.com/zenn-user-upload/cba0bcc5fb9e-20211203.png)


## 参考記事

https://zenn.dev/koron/articles/7cdfd5382f9ee3

https://101010.fun/iot/pro-micro-blink-led.html
