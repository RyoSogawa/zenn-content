---
title: "ReactNative開発環境構築記録（MacOS）"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ReactNative"]
published: false
---

ReactNative の環境を構築したときの流れをつまづきポイントを含めて記録する。

基本的には[公式Doc](https://reactnative.dev/docs/environment-setup)に沿っていく。
開発マシンの OS を選択した上で、。

- Android
- iOS

を選択すると各デバイス用の環境セットアップ方法が確認できる。

※node はインストールされている前提で。

## プロジェクト作成
```
npx react-native init MyApp --template react-native-template-typescript
```

## iOSの開発環境を構築
### watchmanインストール
Facebook 開発のファイルウォッチャー。

```
brew install watchman
```

### Xcodeアプデ
ドキュメントで指定されている設定にはじめからなってたのでそのまま。

### CocoaPods
ここでまず躓いた。
```
sudo gem install cocoapods
```

[CocoaPods公式](https://cocoapods.org/)では ruby を使ったインストールを推奨されていたのでこの方法でインストールしょうとしたがエラーが発生。

```
ERROR:  Error installing cocoapods:
    ERROR: Failed to build gem native extension.
```

Qiita の記事に同様の内容が見つかったのでこの方と同様の方法で解決。

https://qiita.com/mix_dvd/items/aff8dfbe849a3d948715

※このときに `universal-darwin` のバージョン番号が変わっていたので要注意。
log を見ると中のテキストから必要なバージョン番号が見られる。

あとはエラーメッセージを参考に `./ios` 内で `pod install` とかやったりした。

とりあえず iOS は難なくクリア。


## Android
対する Android はかなり苦戦。。

### Java Delvelopment Kit
```
brew install --cask adoptopenjdk/openjdk/adoptopenjdk8
```

### Android Studio
まず Android Studio をセットアップする。

ドキュメントに記載されているライブラリを指定バージョンでインストールしていく。


### zshrc
zshrc に ANDROID 関連のパスを通す。

```
export ANDROID_HOME=$HOME/Library/Android/sdk
export PATH=$PATH:$ANDROID_HOME/emulator
export PATH=$PATH:$ANDROID_HOME/tools
export PATH=$PATH:$ANDROID_HOME/tools/bin
export PATH=$PATH:$ANDROID_HOME/platform-tools
```

### バーチャルデバイスを作成
ここも躓いた。ちゃんと読んでなかっただけだが、iOS はコマンドを実行すればシミュレータが起動するのに対して、Android は先にエミュレータを立ち上げておく必要がある。

>If you have recently installed Android Studio, you will likely need to create a new AVD. Select "Create Virtual Device...", then pick any Phone from the list and click "Next", then select the Q API Level 29 image.

>Click "Next" then "Finish" to create your AVD. At this point you should be able to click on the green triangle button next to your AVD to launch it, then proceed to the next step.


### エミュレータ起動後実行するも動かない
```
yarn android
```

を実行するもなかなか動かない。

```
info Starting the app...
error Failed to start the app.
Error: spawnSync adb ENOENT
    at Object.spawnSync (internal/child_process.js:1041:20)
```

最終的に、AndroidStudio で `/android` を開き、ターミナルで。
```
npx react-native run-android
```

で動いた。。

この方法だと先にエミュレータを起動しておく必要もなさそう。
