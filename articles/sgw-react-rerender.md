---
title: "【社内勉強会資料】React③ 再レンダリングについて"
emoji: "🙆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React" ,"useMemo","useCallback"]
published: false
---

この記事は社内勉強会のために作成しました。
主に React の基本+αの内容について、自分の**超主観的**な理解をシェアしようと思います。

:::message
タイトルの通り内容は**自分の推測**になります。  
間違ってること等あると思うので、指摘や補足等ありましたらやさしく教えてもらえると嬉しいです。
:::

- 第 1 回 ... Type から推測する JSX の基本
- 第 2 回 ... useState と useEffect の中身
- 第 3 回 ... 再レンダリングについて（この記事）
- 未定 ... useRef/useReducer について
- 未定 ... パフォーマンス計測方法

## 今回の会で話す内容
- React のレンダリングが発生するタイミング
- React の再レンダリングを抑制する方法
- `React.memo`, `useMemo`, `useCallback` について

本サンプルプロジェクトは以下記事の内容を大いに踏襲してます。

https://qiita.com/seira/items/9e38204758030cd5442a

## 初期整備

Chrome の React-dev-tools 拡張機能から、Profiler 内の `Highlight~` にチェックを入れておくと再レンダリング対象が分かりやすくなったりします。  
[![Image from Gyazo](https://i.gyazo.com/4be8c4c491842f42971db9d34eb135c3.png)](https://gyazo.com/4be8c4c491842f42971db9d34eb135c3)


とりあえず React のプロジェクトを作ります。
今回は[Vite](https://vitejs.dev/)で。
https://vite.new/react にアクセスするとすぐ始められたりします。

※最終版はこちら。

@[codesandbox](https://codesandbox.io/embed/react-re-render-m9943?fontsize=14&hidenavigation=1&theme=dark)

`src/App.jsx` にてカウンタができてるがボタン以外は不要なので消します。
代わりにログ出力を足します。

```jsx:App.jsx
import {useState} from 'react'
import './App.css'

function App() {
  const [count, setCount] = useState(0)
  console.log('App') // 足す

  return (
    <button type="button" onClick={() => setCount((count) => count + 1)}>
      count is: {count}
    </button>
  )
}

export default App
```


リロードしたりボタンを押すとログに `App` が出力されます。

これはレンダリングが発生したことを示しています（だいたい）。

関数を実行することで描画を更新していると言えるでしょう。

## Reactのレンダリングとは

https://qiita.com/hellokenta/items/6b795501a0a8921bb6b5

> レンダリングとは、現在のPropsとStateを元に、Reactがコンポーネントに対して、それらがどのように見えるべきなのかを尋ねるプロセスです。

> コンポーネントツリー全体からレンダリング出力を収集した後、React はオブジェクトの新しいツリー（「仮想 DOM」と呼ばれることが多い）の差分計算をし、実際の変更すべきDOMのリストを収集します。この差分と計算のプロセスは、"reconcilation"として知られています。

つまり、React は JSX を元に仮想 DOM としてオブジェクトのツリーを形成し、その中で変更すべきコンポーネントを算出して再レンダリングしているということ（曖昧な理解）。

では何をもって変更すべきかどうかを判断しているのかを見ていきます。


## ボタンをコンポーネントにする
`src/Counter.jsx` を作ります。

```jsx:Counter.jsx
const Counter = ({name, count, onClick}) => {
  console.log('Counter' + name)

  return (
    <button type="button" onClick={onClick}>
      Counter{name}: {count}
    </button>
  )
}
export default Counter
```

`App.tsx` から 3 つ呼び出します。

```jsx:App.tsx
import {useState} from 'react'
import './App.css'
import Counter from "./Counter"

function App() {
  const [countA, setCountA] = useState(0)
  const [countB, setCountB] = useState(0)
  const [countC, setCountC] = useState(0)
  console.log('App')

  const incrementA = () => setCountA(prev => prev + 1)
  const incrementB = () => setCountB(prev => prev + 1)
  const incrementC = () => setCountC(prev => prev + 1)

  return (
    <div>
      <Counter name={'A'} count={countA} onClick={incrementA}/>
      <Counter name={'B'} count={countB} onClick={incrementB}/>
      <Counter name={'C'} count={countC} onClick={incrementC}/>
    </div>
  )
}

export default App
```

このときに Counter をクリックするとどのコンポーネントが再レンダリングされるでしょう。

:::details 結果を見る
全コンポーネントが再レンダリングされている。
:::


### なぜこうなるのか
React で再レンダリングが発生するのは、

- props が更新されたとき
- state が更新されたとき

のいずれかです。

https://ja.reactjs.org/docs/optimizing-performance.html#avoid-reconciliation
>コンポーネントの props や state が変更された場合、React は新しく返された要素と以前にレンダーされたものとを比較することで、実際の DOM の更新が必要かを判断します。それらが等しくない場合、React は DOM を更新します。

そして、それが発生したコンポーネントの子コンポーネントは（基本的に）全て再レンダリングされます。


↑の例では、

1. `Counter` の onClick で `App.jsx` の state が更新される
1. `App.jsx` が再レンダリングされる
1. `App.jsx` の子コンポーネントである全ての `Counter` も再レンダリングされる

という具合になります。

## 問題はないが
特に今回のような軽量なコンポーネントでは、レンダリングに対してコストがかかってないので何も気にすることはないです。（←これは結構大事だったりする）

## CounterCを重くする
関数内で `heavyCalc` を実行することでレンダリング速度を下げてみます。

```jsx:App.jsx
import {useState} from 'react'
import './App.css'
import Counter from "./Counter"

// 追加
const heavyCalc = () => {
  let i = 0
  while (i < 1000000) {
    i++
    let j = 0
    while (j < 1000) {
      j++
    }
  }

  return i
}

function App() {
  const [countA, setCountA] = useState(0)
  const [countB, setCountB] = useState(0)
  const [countC, setCountC] = useState(0)
  console.log('App')

  const incrementA = () => setCountA(prev => prev + 1)
  const incrementB = () => setCountB(prev => prev + 1)
  const incrementC = () => setCountC(prev => prev + 1)

  // 追加
  const bigNumber = heavyCalc(countC)

  return (
    <div>
      <Counter name={'A'} count={countA} onClick={incrementA}/>
      <Counter name={'B'} count={countB} onClick={incrementB}/>
      <Counter name={'C'} count={bigNumber + countC} onClick={incrementC}/>
    </div>
  )
}

export default App
```

どちらのボタンを押しても反映までちょっと待つ感じになれば OK。

## パフォーマンス向上へのモチベーション
こうなってくると処理を早くしたいモチベーションが沸いてきましたね！

そこでこの App に対してどういうパフォーマンス改善ができるか考えてみます。

## ①`heavyCalc` は毎度計算しなくていいはず
`ButtonWithState` をよく見てみると `heavyCalc` を実行するのは 1 度だけでいいはずです。
log と同時に走ってるはずなので state を更新する度に実行されている様子。

その結果、このコンポーネントをクリックした際に毎度ちょっとラグが発生しています。

特定のタイミングでだけ再計算されるようにしたいときに使えるのが `useMemo` です。

```diff:ButtonWithState.jsx
-  const bigNumber = heavyCalc(countC)
+  const bigNumber = useMemo(() => heavyCalc(countC), []);
```

第二引数の deps に空配列を指定することで初回のみ処理が走るようになります。
※deps については useEffect 回を参照。

今回のコンポーネントであればこれだけでほぼパフォーマンス問題はクリアされます。
が、コンソールを見ると毎度全ての Counter コンポーネントが再レンダリングされているので、これも制御してみましょう。

## ②`CounterA/B` クリック時には `CounterC` の再レンダリングはされなくてもいいはず
詰まるところ、親コンポーネントが再レンダリングされても子コンポーネントを再レンダリングしない方法ということです。

今回の例でいうと、App コンポーネントの state が更新されたときに、影響のない Counter コンポーネントを再レンダリングしないようにするということです。

### 2-1 stateを子コンポーネントに移動する
最もシンプルな解決策は state をなるべく末端のコンポーネントに移動することでしょう。
大体のケースにおいて、まずはこれを検討するのがいいと思います。

今回の例では例えば B のカウンターを別コンポーネントにしてそっちに state をもたせる方法があります。

```jsx:CounterB.jsx
import {useState} from "react"
import Counter from "./Counter"

const CounterB = () => {
  const [countB, setCountB] = useState(0)

  const incrementB = () => setCountB(prev => prev + 1)

  return <Counter name={'B'} count={countB} onClick={incrementB}/>
}

export default CounterB
```

```diff:App.jsx
- <Counter name={'B'} count={countB} onClick={incrementB}/>
+ <CounterB />
```

こうすることで CounterB をクリックしたときに更新される state は `App`→`CounterB` に変わります。
そのため、CounterB をクリックしても、コンソールに `App` や他のカウンターが出力されなくなります。

しかし、CounterA/C をクリックしたときは全てのコンポーネントが再レンダリングされるでしょう。

### 2-2 `React.memo`
次にメモ化について。

React.memo を使うことでコンポーネントがいつ再レンダリングされるべきなのかをコントロールできるようになります。

::: message
クラスコンポーネントでいうと `shouldComponentUpdate` や `PureComponent` と関係があります。
これから使うことはほぼ無いと思いますが、興味のある方は調べてみてください。
:::

React.memo には省略可能な第二引数があります。
そこでは、props の新旧値を比較して、このコンポーネントが再レンダリングされるべきかどうかを判定するロジックを実装できます。
（※未指定のときは自動的に props が等しいかどうかを判断される。）

:::message
このときに props にオブジェクト型が使われていると React は浅い比較をすることになるので、props が等しいかどうかが正確に判断されなくなってしまいます。
props には極力プリミティブ型を使うのがシンプルでいいでしょう。
※オブジェクト型の対策は最後に。
:::

今回のケースでは props がないのもあって、省略でよさそうです。

export default している箇所に記述するのが楽ですね。

```diff:Counter.jsx
- export default Counter
+ export default memo(Counter)
```

```diff:CounterB.jsx
- export default CounterB
+ export default memo(CounterB)
```

これで OK になりそうですが、実際に再レンダリングが抑止されたのは CounterB のみ。
この違いは何から発生しているのでしょう。

### 2-3 `useCallback`
Counter コンポーネントのほうは props に関数が送られています。

コンポーネント内で定義される関数は、コンポーネントが再レンダリングされたら再生成されます。
同じ関数が props で渡っていても別ものとして扱われるため、メモ化が機能せずに再レンダリングが実行されてしまいます。

そこで `useCallback` の出番です。

increment 関数を useCallback で wrap しましょう。

```js:App.jsx
  // これを
  const incrementA = () => setCountA(prev => prev + 1)
  const incrementB = () => setCountB(prev => prev + 1)
  const incrementC = () => setCountC(prev => prev + 1)

  // こうする
  const incrementA = useCallback(
    () => setCountA(prev => prev + 1),
    [],
  )
  const incrementB = useCallback(
    () => setCountB(prev => prev + 1),
    [],
  )
  const incrementC = useCallback(
    () => setCountC(prev => prev + 1),
    [],
  )
```

これによって値が変化した Counter のみ再レンダリングされるようになったはずです！

## 補足①：useMemoとuseCallbackの関係について
useCallback が useMemo の糖衣構文であることを補足します。

```js
// 以下のコードは全く同じ動作になる
useMemo(() => () => true, [])
useCallback(() => true, [])
```

関数の返り値を扱うのが useMemo で、関数そのものを扱うのが useCallback です。

## 補足②：React.memoにおけるオブジェクト型propsの比較について
デフォルトの React.memo の props の比較には浅い比較（shallow equal）が使われています。

JS におけるオブジェクト型は参照型なので、値が一致していても別物と判断されることがあります。

https://ja.reactjs.org/docs/react-api.html#reactmemo
>デフォルトでは props オブジェクト内の複雑なオブジェクトは浅い比較のみが行われます。

※浅い比較はおそらく `Object.is` のことを指しています。
https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Object/is

```js
var foo = { a: 1 };
var bar = { a: 1 };
Object.is(foo, foo);         // true
Object.is(foo, bar);         // false
```

※ここがピンと来ない人は JS の型について要復習。

https://infoteck-life.com/a0074-js-variable-basic-address/

つまり、全く同じデータを持ったオブジェクトだったとしても、再生成されているものであれば、 React は別のデータと判断し再レンダリングが走ってしまうのです。

### 対策①
 `useMemo` で props に渡す Object をメモ化することでとりあえず解決します。


### 対策②
props に渡すオブジェクトを immutable（不変）にすることでもおそらく解決できます。

https://sbfl.net/blog/2018/09/25/javascript-immutable-programming/

オブジェクトが不変であれば props の新旧比較が同一と判定されて再レンダリングを抑止できるでしょう。

イミュータブルは自力でもある程度実装できますが、 イミュータブルがどうかがちょっと分かりにくい＆コードが見にくくなるのでライブラリを使うことが推奨されています。
有名どこは Facebook 開発の `immutable-js`。

https://immutable-js.com/

```ts
// 利用イメージ
const { Map } = require('immutable');
const map1 = Map({ a: 1, b: 2, c: 3 });
const map2 = map1.set('b', 50);
map1.get('b') + ' vs. ' + map2.get('b'); // 2 vs. 50
```

### 対策③
もう 1 つの対策は React.memo の第 2 引数で deep equal で比較すること。

参照比較と比べて処理時間が長くなる傾向にあります。

業界最速を謳う fast-deep-equal に期待したいですね。

https://github.com/epoberezkin/fast-deep-equal

```jsx
import equal from 'fast-deep-equal'
// deep-equalで比較した結果等しければ再レンダリングしない
export default React.memo(Counter, (prev,next) => equal(prev,next)) 
```
