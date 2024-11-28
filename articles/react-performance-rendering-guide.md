---
title: "優先度順：Reactの再レンダリング最適化ガイド"
emoji: "🚤"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['React', '再レンダリング', 'パフォーマンス']
published: false
---

この記事では、Reactのプロジェクトを実装するにあたり、余計な再レンダリングを抑制することでパフォーマンスを向上させるための施策を**実装の優先度順**に紹介します。

:::message
優先順位はコストと影響を元に主観で設定しています。
:::

## はじめに再レンダリングのルールについて
まずはじめに、Reactの再レンダリングのルールについて簡単におさらい。
Reactのコンポーネントが再レンダリングされるためには

1. コンポーネントの`props`が変更された
2. コンポーネントの`state`が変更された
3. コンポーネントの親コンポーネントが再レンダリングされた

のいずれかの条件を満たす必要があります。

それでは早速抑制する方法を紹介していきます！

## 1. なるべくstateを使わない
stateは再レンダリングを引き起こしたい場面に限定して使いましょう。
再レンダリングが不要なケースでは`useRef`を使って状態を管理することで再レンダリングを抑制できます。

```tsx
import React, { useRef } from 'react';

const Component = () => {
  const hasClicked = useRef(false);
  
  const handleClick = () => {
    if (hasClicked.current) return;
    alert('Alert');
    hasClicked.current = true;
  }

  return (
      <button onClick={handleClick}>1回目だけAlert</button>
  );
};
```

また、算出出来る値はstateを使わずにレンダリング時に計算することも有効です。
`useMemo`を使ってキャッシュする方法と併せて検討しましょう。

## 2. stateを持つコンポーネントを小さくする
stateを持つコンポーネントが再レンダリングの対象となるので、stateを持つコンポーネントを小さくすることで再レンダリングの範囲を抑制できます。
なるべく末端の子コンポーネントにstateを持たせるようにするか、部分的にコンポーネントを分割するのも効果的です。
stateのスコープが狭くなり、凝集度が高まるケースも多いです。

```tsx
// before
const Component = () => {
  const [count, setCount] = useState(0);
  const handleClick = () => setCount(count + 1);

  return (
    <div>
      <button onClick={handleClick}>{count}</button>
      <HeavyComponent /> {/* クリックごとに再レンダリングされてしまう... */}
    </div>
  );
};
```
```tsx
// after
// stateを別コンポーネントに分割
const CountUpButton = () => {
  const [count, setCount] = useState(0);
  const handleClick = () => setCount(count + 1);

  return <button onClick={handleClick}>{count}</button>;
};

const Component = () => {
  return (
    <div>
      <CountUpButton />
      <HeavyComponent />
    </div>
  );
};
```

## 3. useMemo / useCallback
`useMemo`および`useCallback`はeslintのお陰でdeps漏れを検知出来るため安全に導入しやすくておすすめです。

関数コンポーネント内で定義される関数は再レンダリング時に再生成されるため、`useCallback`は積極的に導入していいでしょう。

ただし、`useMemo`については導入が逆効果になるケースもあります。
シンプルなプリミティブ値であればメモ化する恩恵があまりないので、算出処理が複雑な場合に限定して導入すると良いでしょう。

```tsx
const Component = () => {
  const [count, setCount] = useState(0);
  const handleClick = useCallback(() => setCount(count + 1), [count]);

  const bigNumber = useMemo(() => heavyCalc(count), [count]);
  
  return (
    <div>
      <button onClick={handleClick}>{count}</button>
      <p>{bigNumber}</p>
    </div>
  );
};
```

depsにオブジェクト型が指定されるパターンでは意図せぬ再生成が発生することがあります。
その場合はdeepEqualなどを使って比較することで対処できます。

```tsx
import equal from 'fast-deep-equal';

const useDeepCompareMemoizeDeps = <T extends DependencyList[number]>(
  deps: T[]
) => {
  const ref = useRef<T[]>([]);
  if (
    deps.some((dependency, index) => !equal(dependency, ref.current[index]))
  ) {
    ref.current = deps;
  }
  return ref.current;
};

const Component = () => {
  const [user, setUser] = useState({name: 'John'});
  const handleClick = useCallback(() => setUser({name: 'Paul'}), []);

  const deepComparedDeps = useDeepCompareMemoizeDeps([user]);
  const heavyValue = useMemo(() => heavyCalc(user), deepComparedDeps);
  
  return (
    <div>
      <button onClick={handleClick}>{user.name}</button>
      <p>{heavyValue}</p>
    </div>
  );
};
```


## 4. childrenを使ってコンポーネントを分割する
`props.children`の範囲はコンポーネントの再レンダリングの対象外になります。
関数型等のchildrenを使う場合は別で考慮が必要ですが、シンプルな`ReactNode`型であればchildrenに逃がすことで簡単に再レンダリングを抑制出来ます。

```tsx
const Component = ({children}) => {
  const [count, setCount] = useState(0);
  const handleClick = () => setCount(count + 1);

  return (
    <div>
      <button onClick={handleClick}>{count}</button>
      {children} 
    </div>
  );
};

const App = () => {
  return (
    <Component>
      {/* Componentが再レンダリングされてもchildrenのHeavyComponentは再レンダリングされない */}
      <HeavyComponent /> 
    </Component>
  );
};
```


## 5. React.memo
最後に再レンダリング抑制の代表的APIである`React.memo`です。
`React.memo` を使うことでコンポーネントがいつ再レンダリングされるべきなのかをコントロールできるようになります。

`React.memo` には省略可能な第二引数があります。
そこでは、props の新旧値を比較して、このコンポーネントが再レンダリングされるべきかどうかを判定するロジックを実装できます。

:::message
第2引数が未指定のときは自動的に props が等しいかどうかを判断されます。
このときに props にオブジェクト型が使われていると React は**浅い比較**をすることになるので、props が等しいかどうかが正確に判断されなくなってしまいます。
`React.memo`の導入は慎重に行いましょう。
:::

```tsx
const HeavyComponent = React.memo(({count}) => {
  return <p>{heavyCalc(count)}</p>;
});
```

第2引数の比較を拡張することでオブジェクトの等価比較をすることが出来ます。
こちらもコストがかかるため、必要な場面で、必要なpropsにのみ導入するようにしましょう。

```tsx
import equal from 'fast-deep-equal';

const HeavyComponent = React.memo((props) => {
  return <p>{heavyCalc(props)}</p>;
}, (prevProps, nextProps) => {
  return equal(prevProps, nextProps);
});
```

## 最後に
以上、Reactの再レンダリングを最適化するための優先度順ガイドでした。

この中でも特に`React.memo`の導入はリスクやデメリットもあるのた、パフォーマンスの改善が必要になるまでは様子見をするのがオススメです。
また、パフォーマンスを実際に計測しながら導入しましょう。

その他の方法はコストが低く、将来的に`React.memo`を導入する際のコストも下げることが出来るため、早い段階から取り入れていってもよいかと思います。

