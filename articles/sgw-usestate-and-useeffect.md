---
title: "【社内勉強会資料】React②　フックの基本とuseStateとuseEffectの中身推測"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React","reacthook"]
published: false
---

この記事は社内勉強会のために作成しました。
主に React の基本+αの内容について、自分の**超主観的**な理解をシェアしようと思います。

:::message
タイトルの通り内容は**自分の推測**になります。  
間違ってること等あると思うので、指摘や補足等ありましたらやさしく教えてもらえると嬉しいです。
:::

- 第 1 回 ... Type から推測する JSX の基本
- 第 2 回 ... フックの基本と useState と useEffect の中身推測（この記事）
- 第 3 回 ... 再レンダリングについて
- 未定 ... useRef/useReducer について
- 未定 ... パフォーマンス計測方法

## 今回の会で話す内容
- フックの基本的な仕組みとルール
- useState の動き
- useEffect の動き

## はじめに
まずフックについて。

https://ja.reactjs.org/docs/hooks-intro.html

>フック (hook) は React 16.8 で追加された新機能です。state などの React の機能を、クラスを書かずに使えるようになります。

もともとはクラスコンポーネントで複雑に多くの文字をタイプして実装されていたステートやライフサイクルメソッドを関数コンポーネントでもっと簡潔にかけるようにしたものです。

```jsx:before
class Example extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      count: 0
    };
  }

  render() {
    return (
      <div>
        <p>You clicked {this.state.count} times</p>
        <button onClick={() => this.setState({ count: this.state.count + 1 })}>
          Click me
        </button>
      </div>
    );
  }
}
```

```jsx:after
import React, { useState } from 'react';

function Example() {
  const [count, setCount] = useState(0);

  return (
    <div>
       <p>You clicked {count} times</p>
       <button onClick={() => setCount(count + 1)}>
         Click me
       </button>
    </div>
  );
}
```

フックには、

- useState
- useEffect

以外にいくつかの種類がありますが、今回は最もベースとなるこの 2 つのフックについて紹介します。

### フックが行っていること
関数コンポーネントに対して 1 つの格納庫があるようなイメージを自分は持っています。  
そこに値を保持した状態で再実行→再レンダリングをすることで、 state の管理がされているイメージです。

#### 自分のフックに対する格納庫のイメージ

```jsx
// このコンポーネントだと下のテーブルのようにフックは値を格納する
const Example () => {
  const [bool, setBool] = useState(false)
  useEffect(() => {...})
  const [str, setStr] = useState('test')
  const value = useMemo(() => [{...},{...},...])

  //...
}
```

| 関数内で実行された順序 | フックの種類 | 値 |
|:-:|:-:|:-:|
| 1 | useState  |  `false` |
|  2 | useEffect  |  `() => {...}` |
| 3  | useState  |  `'test'` |
| 4  | useMemo  |  `() => [{...},{...},...]` |

フックには ID のようなものがないため、実行順序が唯一の指標になります。
(そのおかげで、フック呼び出し時にわざわざ ID を指定せずに済んでいる)

このためにフックは、

>フックをループや条件分岐、あるいはネストされた関数内で呼び出してはいけません。代わりに、あなたの React の関数のトップレベルでのみ、あらゆる早期 return 文よりも前の場所で呼び出してください。

というルールが設定されているのです。

https://ja.reactjs.org/docs/hooks-rules.html

```jsx
 // 🔴 We're breaking the first rule by using a Hook in a condition
  if (name !== '') {
    useEffect(function persistForm() {
      localStorage.setItem('formData', name);
    });
  }
```

もしこの記法を許してしまうと、実行順序が実行ごとに異なるため、どの値がどのフックのものなのかが分からなくなってしまいます。

## useStateについて
まず大事なこととして、`useState` は関数です。

```jsx
const [count, setCount] = useState(0);
```

関数にステートの初期値を渡すことで、返り値としてステートの値とセッターを受け取ることができます。

上記例で挙げた、以下のようなコンポーネントについて考えます。

```jsx:after
import React, { useState } from 'react';

function Example() {
  const [count, setCount] = useState(0);

  return (
    <div>
       <p>You clicked {count} times</p>
       <button onClick={() => setCount(count + 1)}>
         Click me
       </button>
    </div>
  );
}
```

1. 初回レンダリング時にこのコンポーネント関数は実行され、useState も実行される。
1. `count` である**1番目のフック格納庫**に値がまだないことを確認し、初期値の `0` を格納し `count` として返却する。
1. button をクリックするとセッターが呼ばれ**1番目のフック格納庫**に入っている値を更新する。
1. ステートが更新されるとコンポーネントが再レンダリングされ、useState も実行される。
1. すでに格納庫に値が入っているため、初期化せずに格納庫の値 `1` を `count` として返却する。

おそらくだいたいこんな感じでステートの値は保存され、ステートの更新によって表示が更新されます。


### 余談
#### 関数の返り値をオブジェクトにするべきか、配列にするべきか
ちなみに `useState` の返り値がオブジェクトではなく、配列になっているのはなぜでしょう。

こういった複数の返り値を指定してあげる必要のある関数にとって、オブジェクトと配列の返り値はそれぞれ明確なメリット・デメリットがあります。

|  | 配列 | オブジェクト |
|:-|:-|:-|
| メリット | 呼び出し側で変数名を指定しやすい  | 必要なプロパティだけを抜き取りやすい<br/>デフォルトの変数名を指定しやすい  |
|  デメリット |  返り値の種類が多いときに取捨選択しにくい |  変数名を毎度オリジナルに定義したいときに手間 |


そのため、

- 返り値の数が少ない
- 呼び出し元ではほぼ常に全ての返り値を使用する
  - または値を必ず前から順番に使用する
- 返り値の変数名は毎度定義したい

といった状況であれば**配列**に、

- 返り値の数が多い
- 呼び出し元では必要な値だけを限定して使うことになる
- 返り値の変数名は常に一緒でよい

といった状況であれば**オブジェクト**が向いているでしょう。

複数の値を返す関数やカスタムフックを作る際には、このような基準で選択しています。

#### useStateの初期値に関数を指定する
ちょっとマイナーな機能として、useState の初期値に関数を指定する方法があります。

```js
const [count, setCount] = useState(() => getValue())
```

初期値を求めるときしかこの関数は実行されません。

比較のために、初期値に関数を使わずに上記処理を実装するとこうなります。

```js
const initialCount = getValue()
const [count, setCount] = useState(initialCount)
```

このように記述すると、コンポーネントがレンダリングされる度に `getValue` が実行されることになってしまいます。


## useEffectについて
次に useEffect について。
※基本的にもうこのページを読んでもらったほうがいいです。

https://overreacted.io/ja/a-complete-guide-to-useeffect/

useEffect は React の**副作用フック**と呼ばれ、主にクラスコンポーネントにあったライフサイクルメソッドを実現できるものです。

※ただし、useEffect を使う際に、毎度特定のライフサイクルメソッドの代わりとして使うのは個人的にあまりおすすめしません。
(あくまでも別物なので、 `useEffect` として使うほうがより自然な使い方になるかと)


### ライフサイクルメソッドとは
クラスコンポーネントのライフサイクルメソッドは以下の通りです。

1. componentDidMount
1. shouldComponentUpdate
1. componentDidUpdate
1. componentWillUnmount

この中で `shouldComponentUpdate` 以外の 3 つ（以上のもの）を useEffect は実現できます。

### useEffectの実行タイミング
useEffect に記述した処理はコンポーネントのレンダリングを**ブロックしません**。

例えば、

```jsx
function Example() {
  useEffect(() => alert('hello'))

  return <div>hello</div>
}

function Example2() {
  alert('hello')

  return <div>hello</div>
}
```
このような 2 つのコンポーネントがあったときにどのような違いがあるでしょう。
  
Example のほうは useEffect 内の関数が実行される前（もしくは同時）に return まで実行されます。
それに対して、Example2 のほうは alert が実行されるまで return にたどり着きません。

軽い処理であれば大差ありませんが、重たい処理を useEffect を使わずに呼び出すと処理が完了するまで jsx が return されず、レンダリングされないという動作になってしまいます。

そのため、API キック等を useEffect 内に書き、loading の state 等と併用するのが通例となっています。  
（※React18 からは変わってきそう）

### depsについて
useEffect（や useMemo,useCallback 等）の第二引数は `deps` と呼ばれ、依存関係を表します。
deps に指定された値に変化があるときに useEffect が実行されます。

#### 指定なし
deps なしの useEffect はコンポーネントが再レンダリングされる度に実行されます。

```js
useEffect(() => somefunction())
```

#### 空配列
最もシンプルなケースは空配列を指定することです。

```js
useEffect(() => somefunction(), [])
```

こうすることで初回マウント時のみ実行されるようになり、実質 componentDidMount のライフサイクルメソッドと同等の働きとなります。

なぜそうなるのでしょう。

deps は実行ごとに**前回の値と比較される**ようになっています。

例えば state が更新されてコンポーネントが再レンダリングされたときに、前回の deps の値と再レンダリング時の値を比較して、変化していたら useEffect 内部が実行されます。

そのため、空配列を指定しているとレンダリングの回数やタイミングによらず常に同じ値が入っているため、初回のみ実行されることになるのです。

#### その他なんらかの値
deps に値が指定されると、前述の通り、レンダリングの度に比較され、もし値が変わっていたら useEffect 内部が実行されます。

下記コンポーネントでボタンをクリックした際の各フックの挙動を想像してみます。

```js
function Example() {
  const [count, setCount] = useState(0);
  useEffect(() => console.log(count), [count])

  return (
       <button onClick={() => setCount(count + 1)}>
         Click me
       </button>
  );
}
```
##### ①初回マウント時

|  | 1つ目のフック | 2つ目のフック |
|:-:|:-:|:-:|
| フックの種類  |  state |  effect |
| 値  |  `0` | `() => console.log(0)`  |
| deps  | -  |  `0` |
| 実行  | -  |  初回なので実行される |

##### ②ボタンクリック時

|  | 1つ目のフック | 2つ目のフック |
|:-:|:-:|:-:|
| フックの種類  |  state |  effect |
| 値  |  `1` | `() => console.log(1)`  |
| deps  | -  |  `1` |
| 実行  | -  |  depsの値が前回と異なるので実行される |


だいたいこうなると思います。


#### 関数
関数を deps に指定する場合、期待している動作とならない可能性があります。

```js
const Example () => {
  const func = (name) => console.log(name)
  useEffect(() => func('John'), [func])
  // ...
}
```

このコンポーネントの useEffect が実行されるタイミングはいつになるでしょう。
..。

...

...

答えは **「再レンダリングの度に実行される」** です。

コンポーネント内部で定義した関数は再レンダリングの度に毎度再生成されていて、新しいものとして扱われます。  
そのため、deps が変化したと判断されてエフェクトが発火してしまうのです。

対策の 1 つはコンポーネントの外部で関数を定義すること。
そしてもう 1 つは `useCallback` です。

useCallback でラップした関数は useCallback の deps が更新されなければ同じものが再利用されます。useEffect の deps に指定しても、関数が再利用されていれば、同一のものとみなされて useEffect の発火を防ぎます。

```js
const Example () => {
  const func = useCallback((name) => console.log(name), [])
  useEffect(() => func('ryu'), [func])
  // ...
}
```

関数だけでなく、オブジェクト型の変数でも同様のことが起きるため、コンポーネント内部で定義を書くのは必要最低限にするのが基本的にいいと思います。
