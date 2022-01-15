---
title: "【社内勉強会資料】React①　Typeから推測するJSXの基本"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React","JSX"]
published: false
---

この記事は社内勉強会のために作成しました。
主に React の基本+αの内容について、自分の**超主観的**な理解をシェアしようと思います。

:::message
タイトルの通り内容は**自分の推測**になります。  
間違ってること等あると思うので、指摘や補足等ありましたらやさしく教えてもらえると嬉しいです。
:::

- 第 1 回 ... Type から推測する JSX の基本（この記事）
- 第 2 回 ... useState と useEffect の中身
- 第 3 回 ... 再レンダリングについて
- 未定 ... useRef/useReducer について
- 未定 ... パフォーマンス計測方法


## JSXとは
まず公式によく分かるドキュメントがあるので必読です。

https://ja.reactjs.org/docs/jsx-in-depth.html

JSX は `React.createElement` の糖衣構文とのこと。

### React.createElementとは

https://ja.reactjs.org/docs/react-api.html#createelement

```js
React.createElement(
  type,
  [props],
  [...children]
)
```
こういった関数になります。

例えば、

```jsx
<div className="sidebar" />

// ↓と同義
React.createElement(
  'div',
  {className: 'sidebar'}
)
```

```jsx
<MyButton color="blue" shadowSize={2}>
  Click Me
</MyButton>

// ↓と同義
React.createElement(
  MyButton,
  {color: 'blue', shadowSize: 2},
  'Click Me'
)
```

こんな感じで、普段 HTML のように記述している JSX は裏で`React.createElement`を実行しています。

こんな書き方では HTML の構造がとてもよく分からないので、JSX という糖衣構文を使って簡単に分かりやすく記述**させていただいている**のです。


ここで大事なのは、

- 既存の HTML タグは小文字で開始する
- React コンポーネントは大文字で開始する

というルール。  
React 側ではタグの開始が大文字か小文字かによってそれが HTML タグなのか React コンポーネントなのかを判別しています。

## Typeを見てみる
React における JSX 周りに使われる Type はだいたい以下の通り。（ざっくり）

- `ReactElement`
- `JSX.Element`
- `ReactChild`
- `ReactNode`

下記記事が関係性を画像化しててわかりやすかったので併せてみていただきたい。

https://dackdive.hateblo.jp/entry/2019/08/07/090000

それぞれの関係を見ていくと React の理解がちょっと進みます。

### ReactElement
```ts
interface ReactElement<P = any, T extends string | JSXElementConstructor<any> = string | JSXElementConstructor<any>> {
        type: T;
        props: P;
        key: Key | null;
    }
```

どうやら JSX.Element のコンストラクタである型のようです。
パラメータとして、JSX の `type` と `props` の型を指定できるみたい。
※先述の `React.createElement` 参照。

`JSXElementConstructor` が `string | JSXElementConstructor<any>` となっています。おそらく、`string` は HTML タグ、`JSXElementConstructor<any>` は React コンポーネントのことでしょう。

また、`type`,`props` と並んで `key` があり、key パラメータが React にとって特別なものであることが分かります。

(ちなみに Key 型はこんな感じでした)

```ts
type Key = string | number;
```

### JSX.Element

```ts
namespace JSX {
        interface Element extends React.ReactElement<any, any> { }
         //..
}
```
namespace 内が膨大なので Element の部分だけを抽出しています。

これはどうやら `ReactElement` の 2 つのパラメータに any を渡しただけの型のよう。

### ReactChild
※children とは違う。

```ts
type ReactChild = ReactElement | ReactText;
// type ReactText = string | number;
```

ReactElement 以外に string と number を受け付ける値のようです。

### ReactNode
最後に ReactNode 型。  
`props.children` の型でもある、とても大事なものです。

```ts
type ReactNode = ReactChild | ReactFragment | ReactPortal | boolean | null | undefined;
```

ReactFragment と ReactPortal の型はあとで見ますが、これまでの情報を組み合わせると、

```ts
ReactElement | string | number | boolean | null | undefined
```

だいたいこういうことでしょう。
結構幅広く型を受け付けているように見えるが、これは JSX の以下の仕様に則ったものです。

https://ja.reactjs.org/docs/jsx-in-depth.html#booleans-null-and-undefined-are-ignored

>真偽値、null、undefined は無視される
true と false、null、そして undefined は子要素として渡すことができます。これらは何もレンダーしません。

つまり、↑の型でいうところの `boolean | null | undefined` の部分は受け付けるだけで実際にレンダリングに使われることはないのです。
この仕様が生きるのは JSX 内の分岐を書くとき。

```tsx
<div>
  {condition && <p>text</p>}
</div>
```
たとえばこういう条件付きレンダーを書くことは多いと思いますが、`condition` が falsy な値のときはどうなるのかを考えます。

代表的な js の falsy 値は `'' | 0 | false | null | undefined` あたり。

falsy なときは `&&` の左辺値が出力されることになるので、それぞれ見ていくとこうなります。

```tsx
// ''
<div></div>

// 0
<div>{0}</div>

// false
<div>{false}</div> 

// null
<div>{null}</div>

// undefined
<div>{undefined}</div>
```

そしてこのとき、上述した `真偽値、null、undefined は無視される` というルールから、下 3 つのものは全て `<div></div>` と変換されることになります。

この仕様によって、条件付きレンダーがとても簡単に記述できるのです。

（もしこの仕様がなければ、常に三項演算子を使う必要があるでしょう。）

#### おまけ　ReactFragment

ちなみに、ReactNode 型に含まれている型の中で、ここまで触れなかった `ReactFragment` と `ReactPortal` 型は下記のようになっています。

```ts
interface ReactNodeArray extends Array<ReactNode> {}
type ReactFragment = {} | ReactNodeArray;
interface ReactPortal extends ReactElement {
  key: Key | null;
  children: ReactNode;
}
```

`ReactPortal` 型はほとんど ReactElement のようですが、`ReactFragment` は少しややこしそう。

どうやら `ReactNode` の配列または空オブジェクトの型のよう。これはどういうことなのでしょう。


普段 JSX を書く場合、必ず大外の要素が 1 つでなければなりません。

```tsx
// OK
const jsx: JSX.Element = (
  <div>
    <div>test</div>
  </div>
)

// NG
const jsx: JSX.Element = (
  <div>test</div>
  <div>test2</div>
)
```

しかし、複数の JSX を並列に指定できる場所が（自分が思いつく限り）2 つだけあります。

それが、fragment と children です。

```tsx
const fragment: JSX.Element = (
  <>
    <div>test</div>
    <div>test2</div>
  </>
)

const children: JSX.Element = (
  <div>
    <div>test</div>
    <div>test2</div>
  </div>
)
```

上記要素が並列となっている、

```tsx
    <div>test</div>
    <div>test2</div>
```

の箇所は、配列として JS では扱われているのでしょう。

※先述の React.createElement でいうと、children が配列で指定されているのもこれが理由だと思います。

```js
React.createElement(
  type,
  [props],
  [...children]
)
```

そしてこの配列部分の型を表しているのが ReactFragment と考えていいでしょう。（おそらく）

※ちなみに React では配列を返却できます。

```jsx
render() {
  // リスト化するだけのために要素を用意する必要はありません！
  return [
    // key 属性を書き忘れないようにしてください :)
    <li key="A">First item</li>,
    <li key="B">Second item</li>,
    <li key="C">Third item</li>,
  ];
}
```
https://ja.reactjs.org/docs/jsx-in-depth.html#jsx-children


## まとめ
以上、JSX の基本を推測した内容でした。

React のイメージが少しでも具体的になったらいいなと思います。

このような複雑な処理を人に感じさせずに HTML のように書くことができる JSX はとてもすごい技術だなと思いました。（感想）
