---
title: "優先度順：Reactの再レンダリング最適化ガイド"
emoji: "🚤"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['React', '再レンダリング', 'パフォーマンス']
published: true
published_at: 2024-12-06 07:00
publication_name: any_dev
---
こんにちは！
any株式会社でプロダクトチームに所属しているVPoE/エンジニアの十川です！

この記事は、any Product Team Advent Calendar2024　6日目の記事になります。
https://qiita.com/advent-calendar/2024/anyinc

この記事では、React のプロジェクトを実装するにあたり、余計な再レンダリングを抑制することでパフォーマンスを向上させるための施策を個人的な優先度順に紹介します。
PRレビューの観点等に参考にしていただければ幸いです。

:::message
優先順位はコストと影響を元に主観で設定しています。
:::

## はじめに再レンダリングのルールについて
まずはじめに、React の再レンダリングのルールについて簡単におさらい。
React のコンポーネントが再レンダリングされるためには

1. コンポーネントの `props` が変更された
2. コンポーネントの `state` が変更された
3. コンポーネントの親コンポーネントが再レンダリングされた

のいずれかの条件を満たす必要があります。

それでは早速抑制する方法を紹介していきます！

## 1. なるべくstateを使わない
state は再レンダリングを引き起こしたい場面に限定して使いましょう。
再レンダリングが不要なケースでは `useRef` を使って値を管理することで再レンダリングを抑制できます。

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

また、算出出来る値は state を使わずにレンダリング時に計算することも有効です。
`useMemo` を使ってキャッシュする方法と併せて検討しましょう。

## 2. stateを持つコンポーネントを小さくする
state を持つコンポーネントが再レンダリングの対象となるので、state を持つコンポーネントを小さくすることで再レンダリングの範囲を抑制できます。
なるべく末端の子コンポーネントに state を持たせるようにするか、部分的にコンポーネントを分割するのも効果的です。
結果的に state のスコープが狭くなり、凝集度が高まるケースも多いです。

```tsx:before
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
```tsx:after
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
`useMemo` および `useCallback` は eslint のお陰で deps 漏れを検知出来るため安全に導入しやすくておすすめです。
単体で再レンダリングを抑制する効果はあまり見込めませんが、将来的に `React.memo` を導入しやすくなったり、再レンダリング時のコストが下がるケースもあったりします。

ただし、導入が逆効果になるケースもあります。
特にシンプルなプリミティブ値であればメモ化する恩恵があまりないので、算出処理が複雑な場合に限定して導入すると良いでしょう。

```tsx
const Component = () => {
  const [count, setCount] = useState(0);
  const handleClick = useCallback(() => setCount(prev => prev + 1), []);
  const bigNumber = useMemo(() => heavyCalc(count), [count]);
  
  return (
    <div>
      <button onClick={handleClick}>{count}</button>
      <p>{bigNumber}</p>
    </div>
  );
};
```

deps にオブジェクト型が指定されるパターンでは意図せぬ再生成が発生することがあります。
その場合は deepEqual などを使って比較することで対処できます。
※コメントをいただき修正しました。

```tsx
import equal from 'fast-deep-equal';

const useDeepCache = <T extends DependencyList[number]>(dependency: T) => {
  const ref = useRef<T>(dependency);
  if (!equal(dependency, ref.current)) {
    ref.current = dependency;
  }
  return ref.current;
};

const Component = () => {
  const [user, setUser] = useState({name: 'John'});
  const handleClick = useCallback(() => setUser({name: 'Paul'}), []);

  const memoizedUser = useDeepCache(user);
  const heavyValue = useMemo(() => heavyCalc(memoizedUser), [memoizedUser]);
  
  return (
    <div>
      <button onClick={handleClick}>{user.name}</button>
      <p>{heavyValue}</p>
    </div>
  );
};
```

コストの判断を毎度正確には出来ないと思いますので、プロジェクトにおけるメモ化のルールを明確にして実装者による差がなるべく出ないように出来ると良いかと思います。


## 4. childrenを使ってコンポーネントを分割する
`props.children` の範囲はコンポーネントの再レンダリングの対象外になります。
関数型等の children を使う場合は別で考慮が必要ですが、シンプルな `ReactNode` 型であれば children に逃がすことで簡単に再レンダリングを抑制出来ます。

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
最後に再レンダリング抑制の代表的 API である `React.memo` です。
`React.memo` を使うことでコンポーネントがいつ再レンダリングされるべきなのかをコントロールできるようになります。

`React.memo` には省略可能な第二引数があります。
そこでは、props の新旧値を比較して、このコンポーネントが再レンダリングされるべきかどうかを判定するロジックを実装できます。

:::message
第二引数が未指定のときは自動的に props が等しいかどうかを判断されます。
このときに React は**浅い比較**をすることになるので、props にオブジェクト型が使われていると props が等しいかどうかが正確に判断されなくなってしまいます。
:::

```tsx
const HeavyComponent = React.memo(({count}) => {
  return <p>{heavyCalc(count)}</p>;
});
```

第二引数の比較を拡張することでオブジェクトの等価比較をすることも出来ます。
こちらもコストがかかるため、必要な場面で、必要な props にのみ導入するようにしましょう。

```tsx
import equal from 'fast-deep-equal';

const HeavyComponent = React.memo((props) => {
  return <p>{heavyCalc(props)}</p>;
}, (prevProps, nextProps) => {
  return equal(prevProps, nextProps);
});
```

## 最後に
以上、React の再レンダリングを最適化するための個人的な優先度順ガイドでした。

この中でも特に `React.memo` の導入はリスクやデメリットもあるので、パフォーマンスの改善が必要になるまでは様子見をするのがオススメです。
また、パフォーマンスを実際に計測しながら導入しましょう。

その他の方法はコストが低く、将来的に `React.memo` を導入する際のコストも下げることが出来るため、早い段階から取り入れていってもよいかと思います。

最後までお読みいただきありがとうございました！
