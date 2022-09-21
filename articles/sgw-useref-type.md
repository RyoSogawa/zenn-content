---
title: "useRefの種類"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "hooks"]
published: false
---

useRefには2種類の用途がある。
https://ja.reactjs.org/docs/hooks-reference.html#useref


1. DOMの参照
2. 値の保持

である。

定義するタイミングで使い分ける。

```ts
// React.MutableRefObject<undefined>
const ref1 = useRef();

// React.MutableRefObject<null>
const ref2 = useRef(null);

// React.MutableRefObject<HTMLDivElement | undefined>
const ref3 = useRef<HTMLDivElement>();

// React.RefObject<HTMLDivElement>
const ref4 = useRef<HTMLDivElement>(null);
```

上記の通り、
- 1, 2, 3 は`MutableRefObject`
- 4 は`RefObject`

型と推論される。

この`RefObject`がDOMの参照に使うものであり、その他が値の保持に使うものである。


## 特徴1
Mutableでない`RefObject`型のrefは値を代入することができない。

```ts
ref4.current = null; 
// Cannot assign to 'current' because it is a read-only property.
```

DOMの参照に使うrefをユーザーが変更する必要はないためreadonlyであることは自然だろう。

## 特徴2

初期値に`null`を指定していない 1,3 のパターンはDOM要素のrefに指定するとエラーになる

```tsx
<div ref={ref1} />
// Type 'MutableRefObject<undefined>' is not assignable to type 'LegacyRef<HTMLDivElement> | undefined'.

<div ref={ref3} /> 
// Type 'MutableRefObject<HTMLDivElement | undefined>' is not assignable to type 'LegacyRef<HTMLDivElement> | undefined'.
```

## まとめ
DOM要素の参照に使う場合はちょっと面倒だが
```ts
const ref = useRef<HTMLDivElement>(null);
```
のように、

- Element型の指定
- 初期値に`null`

を指定しよう！

※最後に今回の記事の内容をcodesandboxに作成したので実際に触ってみたいときにどうぞ。
@[codesandbox](https://codesandbox.io/embed/useref-test-jsi8rb?fontsize=14&hidenavigation=1&theme=dark)

## 参考文献
https://zenn.dev/berlysia/articles/624bc1aaffda58#%E3%82%A4%E3%83%B3%E3%82%B9%E3%82%BF%E3%83%B3%E3%82%B9%E5%A4%89%E6%95%B0%E4%BB%A3%E3%82%8F%E3%82%8A%E3%81%AB%E4%BD%BF%E3%81%86%E3%81%A8%E3%81%8D