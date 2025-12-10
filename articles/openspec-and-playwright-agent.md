---
title: "OpenSpecで作成した仕様書を元にPlaywrightAgentにテストコードを更新してもらう"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['PlaywrightAgent', 'OpenSpec', 'E2Eテスト']
published: true
published_at: 2025-12-13 07:00
publication_name: any_dev
---

こんにちは！any 株式会社でプロダクトチームに所属しているエンジニアの十川です。
この記事は、any プロダクトチーム Advent Calendar 2025 シリーズ 1 の 13 日目の記事になります。
https://qiita.com/advent-calendar/2025/anyinc

## はじめに
どこもかしこも AI 一色な 2025 年でしたが、AI を使った開発に自分は正直まだもう一歩馴染んでいません。
キーボードを叩くことそのものが好きな自分としては、AI と会話する文章ではなくコードそのものを書きたい気持ちもあります。

そんな中、テストをすることに大きな手間と負担を感じていた僕はなんとかテストを AI に任せられないかと考えていました。
そこで、OpenSpec と PlaywrightAgent を組み合わせて、仕様書の変更に伴うテストコードの更新を自動化できないか試してみることにします。

### 作成するアプリについて
題材として、ちょうど作ろうと思っているものがあったのでそれを使います。
ギターエフェクターである Zoom の MultiStomp シリーズのエフェクターは、MIDI コントローラーを接続すると内部の操作を MIDI メッセージで制御できるようになります。
その設定を手軽に行うためのツールを実装してみます。

※ 本記事で扱うツールは ZOOM 社が提供する公式ソフトウェアではなく、著者による個人開発の技術検証です。

## 開発手順
### OpenSpecを使った開発フロー

まず OpenSpec をインストールします。

https://github.com/Fission-AI/OpenSpec

```bash
$ npm install -g @fission-ai/openspec@latest

$ cd my-project

$ openspec init
```

利用する AI サービスを選択します。今回は Claude Code のみを選択しました。
すると以下のように表示されます。

```bash
Use `openspec update` to refresh shared OpenSpec instructions in the future.

Next steps - Copy these prompts to Claude Code:
────────────────────────────────────────────────────────────
1. Populate your project context:
   "Please read openspec/project.md and help me fill it out
    with details about my project, tech stack, and conventions"

2. Create your first change proposal:
   "I want to add [YOUR FEATURE HERE]. Please create an
    OpenSpec change proposal for this feature"

3. Learn the OpenSpec workflow:
   "Please explain the OpenSpec workflow from openspec/AGENTS.md
    and how I should work with you on this project"
─────────────────────────────────────────────────────────
```

書かれている通り
> Please read openspec/project.md and help me fill it out with details about my project, tech stack, and conventions

と Claude Code に投げるとプロジェクトの概要を会話形式で確認されました。

```bash
・プロジェクトの目的は何ですか？（例：ZOOM MultiStompをPC/スマホから制御するアプリ） → 自由入力
· 使用する技術スタックは何ですか？ → TypeScript + React
· どのZOOM製品をサポートする予定ですか？ → 複数機種対応
```

:::details プロジェクトの目的に関する返答内容
midi コントローラでマルチエフェクターの操作をするための信号を生成するアプリ
生成された信号は文字列で表示され、ワンタッチでクリップボードにコピーできる
指定できる操作は以下
1〜6 番目のエフェクタの ON
F0 52 00 6E 64 20 00 aa 00 01 00 00 00 00 F7
1〜6 番目のエフェクタの OFF
F0 52 00 6E 64 20 00 aa 00 00 00 00 00 00 F7
1〜6 番目のエフェクタを表示
F0 52 00 6E 64 20 00 64 01 aa 00 00 00 00 F7
aa の箇所に 00〜05 の値が入り、エフェクタの番号を指定する
これらを複数個自由に組み合わせられる

開発環境について
pnpm
react
typescript
vite
shadcn-ui
tailwindcss
を使ったシンプルなクライアント web アプリ

デザインはダークモードの SP を前提に
:::

これらの情報を元に `openspec/project.md` が生成されました。

:::details 生成された `openspec/project.md` の内容
````markdown
# Project Context

## Purpose

MIDIコントローラでZOOMマルチエフェクター（MultiStomp等）を操作するためのMIDI SysExメッセージを生成するWebアプリケーション。

主な機能:
- エフェクター1〜6のON/OFF制御メッセージ生成
- エフェクター1〜6の表示切り替えメッセージ生成
- 複数の操作を自由に組み合わせ可能
- 生成したメッセージをワンタッチでクリップボードにコピー

## Tech Stack

- **言語**: TypeScript
- **フレームワーク**: React
- **ビルドツール**: Vite
- **パッケージマネージャ**: pnpm
- **UIコンポーネント**: shadcn/ui
- **スタイリング**: Tailwind CSS

## Project Conventions

### Code Style

- 早期リターンを優先し、インデントを浅く保つ
- コメントは最小限に抑える（複雑な処理や意図が明確でない場合のみ）
- TypeScriptの型を活用し、型安全性を重視

### Architecture Patterns

- クライアントサイドのみのシンプルなSPA
- コンポーネントベースのUI設計
- shadcn/uiのコンポーネントを活用

### Testing Strategy

未定

### Git Workflow

未定

## Domain Context

### MIDI SysExメッセージフォーマット

ZOOM MultiStomp向けのSysExメッセージ:

```
エフェクターON (aa = 00〜05でエフェクター番号を指定):
F0 52 00 6E 64 20 00 aa 00 01 00 00 00 00 F7

エフェクターOFF:
F0 52 00 6E 64 20 00 aa 00 00 00 00 00 00 F7

エフェクター表示切り替え:
F0 52 00 6E 64 20 00 64 01 aa 00 00 00 00 F7
```

- `F0`: SysEx開始
- `52`: ZOOMのメーカーID
- `F7`: SysEx終了
- `aa`: エフェクター番号（00〜05 = 1〜6番目）

## Important Constraints

- ダークモード専用
- モバイル（スマートフォン）ファーストのUI設計
- クライアントサイドのみで動作（バックエンド不要）

## External Dependencies

- ZOOM MultiStompシリーズ（MS-50G、MS-60B、MS-70CDR等）
- ZOOM MultiStomp Plusシリーズ（MS-50G+、MS-60B+等）
- その他ZOOM製マルチエフェクター
````
:::

ここまでくれば開発を開始できそうです。

ちなみに OpenSpec の init 時に Claude Code には以下のコマンドが作成されます。
```bash
  /openspec:proposal     Scaffold a new OpenSpec change and validate strictly. (project)
  /openspec:apply        Implement an approved OpenSpec change and keep tasks in sync. (project)
  /openspec:archive      Archive a deployed OpenSpec change and update specs. (project)
```
これらのコマンドを使って開発を進めていきます。

まずは開発基盤を整えるために以下のように投げました。
```bash
 /openspec:proposal プロジェクトのベースを実装 
```

すると `/openspec/changes/[project-name]` にいくつかの資料が作成されます。

```
openspec/changes/scaffold-project-base
├── proposal.md
├── specs
│   ├── clipboard-copy
│   │   └── spec.md
│   └── sysex-generation
│       └── spec.md
└── tasks.md
```

そして最後に `npx openspec validate` が実行され、それぞれの資料がチェックされます。 

内容をチェックし、問題なければ以下のコマンドで実装を開始します。

```bash
/openspec:apply scaffold-project-base
```

気になった点をいくつか直してもらいながら実装が完了したので、チケットをアーカイブします。

```bash
/openspec:archive scaffold-project-base
```

これで一連の操作が完了です。

### PlaywrightAgentを使ったE2Eテストの実装
続いて PlaywrightAgent を使って E2E テストを実装してみます。
https://playwright.dev/docs/test-agents

PlaywrightAgent は主に以下の機能を提供します。
要はテストを自動生成・修復してくれるエージェント群です。
- 🎭 planner ... explores the app and produces a Markdown test plan
- 🎭 generator ... transforms the Markdown plan into the Playwright Test files
- 🎭 healer ... executes the test suite and automatically repairs failing tests

まずは PlaywrightAgent/Playwright をインストールします。
```bash
# Claude Code用のPlaywrightAgent
$ npx playwright init-agents --loop=claude
$ pnpm create playwright
✔ Where to put your end-to-end tests? · e2e
✔ Add a GitHub Actions workflow? (Y/n) · true
✔ Install Playwright browsers (can be done manually via 'pnpm exec playwright install')? (Y/n) · true
```

生成された agent を使ってテスト計画書を作成します。
```
> @agent-playwright-test-planner 現在のアプリ全体のテスト計画書を作成してください
...
 テスト計画が完成しました！

  作成されたテスト計画

  ファイル: specs/app.test-plan.md

  テストスイート構成（全9スイート、26テストケース）

  | スイート          | テスト数 | 内容               |
  |---------------|------|------------------|
  | 1. 初期状態       | 1    | アプリの初期表示確認       |
  | 2. オペレーション追加  | 2    | 初回・複数追加          |
  | 3. 操作タイプ選択    | 3    | ON/OFF/Display切替 |
  | 4. エフェクター番号選択 | 4    | 番号1-6の選択         |
  | 5. オペレーション削除  | 4    | 単体・先頭・中間・末尾の削除   |
  | 6. 全クリア       | 2    | 単体・複数のクリア        |
  | 7. SysEx出力表示  | 5    | 各タイプの出力形式確認      |
  | 8. クリップボードコピー | 4    | コピー・フィードバック確認    |
  | 9. 複雑なワークフロー  | 2    | E2Eシナリオ          |

  各テストケースには具体的なステップと期待結果が記載されています。次のステップとしてテストコードの生成も可能です。 
```

続いてテストコードを生成します。
```
> @agent-playwright-test-generator specs/app.test-plan.md
```

たくさん生成されました。しかし実行するとパスしないものもあります。
Healer に修復をお願いしましょう。
```
> @agent-playwright-test-healer
```

プロンプトを入れずに実行してみましたが、修復を始めてくれました。
主にセレクタの修正と test の設定を行ってくれたようです。

## 仕様変更
**ここからが本題です。**

機能を追加していきます。OpenSpec でドキュメントを書いてもらい(proposal)、実装(apply)してもらいました。

こんなとき、当然ですがテストも変更が必要です。
従来であれば手動でテストコードを修正する必要がありますが、PlaywrightAgent を使えば自動でやってくれます。

そして、OpenSpec を使って作成された仕様書を渡すことで、テスト計画書を簡単に更新してもらえるかもしれません。
そのまま投げてみましょう。

```
> @agent-playwright-test-planner @openspec/changes/add-tuner-volume-tempo の仕様書を元にテスト計画書を更新してください 
```

このタイミングでテストファイルが大きくなりそうだったので分割をしてもらいました。これも playwright-agent がやってくれます。

```
> @agent-playwright-test-planner 今後も仕様が変更されていくことを考えるとテストプランのファイルを分割したほうがよいですか？
```

テストプランファイルを分割したので既存のテストコードもなにか手を入れたほうがいいかもしれません。
generator に質問したところ、以下のように返答がありました。

```
> @agent-playwright-test-generator specs/test-plans/** に各テストファイルを分割しました。テストファイルの構造は変更しますか？

⏺ 現在の構造を比較します：

  | テストプラン                    | e2eディレクトリ                                         | 状態     |
  |---------------------------|---------------------------------------------------|--------|
  | 01-initial-state          | e2e/initial-state.spec.ts                         | ✓ 実装済み |
  | 02-operation-management   | e2e/add-operation/, delete-operation/, clear-all/ | ✓ 実装済み |
  | 03-effector-operation     | e2e/operation-type/, effect-number/               | ✓ 実装済み |
  | 04-tuner-operation        | e2e/tuner/                                        | ✗ 未実装  |
  | 05-patch-volume-operation | e2e/patch-volume/                                 | ✗ 未実装  |
  | 06-tempo-operation        | e2e/tempo/                                        | ✗ 未実装  |
  | 07-sysex-output           | e2e/output-display/                               | ✓ 実装済み |
  | 08-clipboard              | e2e/clipboard/                                    | ✓ 実装済み |
  | 09-complex-workflows      | e2e/workflows/                                    | ✓ 実装済み |

  提案: 現在のe2e構造は機能別に整理されており良好です。変更は不要で、未実装の3つのディレクトリを追加するだけで済みます：

  - e2e/tuner/ (3テスト)
  - e2e/patch-volume/ (3テスト)
  - e2e/tempo/ (3テスト)

  これらのテストを生成しますか？
```

プランに対して実装済みのテストと未実装のテストをそれぞれ洗い出してくれました。
今後はプランファイルに手を入れると、テストコードの変更が必要な箇所を洗い出すこともしてもらえそうです。


## おわりに
以上、OpenSpec と PlaywrightAgent を組み合わせた開発フローの紹介でした。

実際に生成されたツールとリポジトリは以下になります。もし ZOOM の MultiStomp シリーズをお使いの方がいればぜひ使ってみてください。
https://github.com/RyoSogawa/zoom-multistomp-commander

https://zoom-multistomp-commander.vercel.app/

Playwright Agent のようなツールを使うことで、今後は日本語のドキュメントとしてテスト仕様を管理し、テストコードは自動生成・修復してもらう流れが作れるかもしれません。

PlaywrightAgent を使う上での tips 等も今後まとめていきたいと思います。

最後までお読みいただきありがとうございました！
