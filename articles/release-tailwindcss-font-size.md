---
title: "TailwindCSSでフォントサイズをpxToRemでマニュアル指定するプラグインを作った"
emoji: "💤"
type: "tech" # tech: 技術記事 / idea: アイデア 
topics: ["npm","tailwindcss","plugin"]
published: true
---

## 作ったもの
https://www.npmjs.com/package/tailwindcss-font-size

## 使い方
以下の 2 種類のクラス名が有効になります。
- `fsz-{n}px`
- `fsz-{n}ptr`

`px` は指定した数値をそのままピクセルで指定。
`ptr` は `px to rem` の略で、指定した数値をピクセルから rem に変換したフォントサイズがあたります。

```html
<!--ベースフォントサイズが16pxの場合-->
<p class="fsz-18px">font-size:18px</p>
<p class="fsz-18ptr">font-size:1.125rem</p>
```

### 設定
以下の 3 項目が設定できます。

```js:tailwind.config.js
module.exports = {
  theme: {
    // ...
  },
  plugins: [
    require('tailwindcss-font-size')({ 
      baseSize: 16, // ベースフォントサイズ
      minSize: 10, // 最低フォントサイズ
      maxSize: 128  // 最高フォントサイズ
    }),
    // ...
  ],
}
```

## 解決したかったこと

もともと TailwindCSS が好きでよく使っているのですが、少しかゆいのがフォントサイズの指定。

https://tailwindcss.com/docs/font-size

プリセットの値は、

- xs
- sm
- base
- lg
- xl~9xl

で基本的に問題は無いのですが、こと Web 制作でデザイナーのかたからいただくデザインではだいたいもっと細かく指定されています。   
（感覚的には特に 10px ~ 20px の間はだいたい使われている。）

そうなってくると別途クラスを用意するか、JIT で個別指定することになるのですが、`rem` を使うことが多い Web のフォントサイズを都度指定するのは結構めんどくさい。。！

```html
<!-- TailwindCSS JIT -->
<p class="text-[22px]">🥳 JIT素晴らしい</p>
<p class="text-[1.375rem]">🧮 こういうのは計算が必要...</p>
```

これを避けるために、本プラグインを開発しました。

## ※プラグインのインストールが面倒な方へ
以下のコードでもだいたい同じことができます。
`100`/`10`/`16` の箇所を適宜変更してください。

```js:tailwind.config.js
const fontSize = Object.fromEntries(
  [...Array(100)].map((_, index) => { 
    const px = index + 10
    return [`${px}ptr`, `${px / 16}rem`]
  })
)

module.exports = {
  // ...
  extend: {
    fontSize,
    // ...
  }
}
```

```html:生成されるクラス
<p class="text-10ptr">font-size:0.625rem</p>
```

## 最後に
最後にリポジトリのリンクを貼っておきます。

https://github.com/RyoSogawa/tailwindcss-font-size

Issue や PR 歓迎です。
使ってみて便利でしたらスター⭐️をいただけると嬉しいです！
