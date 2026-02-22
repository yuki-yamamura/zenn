---
title: "私の好きなClaude Codeの使い方"
emoji: "🍓"
type: "tech"
topics:
  - "claudecode"
  - "zennfes2025ai"
published: true
published_at: "2025-08-31 18:19"
publication_name: "frontendflat"
---

Claude Codeを使い始めて4ヶ月になりますが、使い方も日々アップデートされていく感覚があります。この記事は、2025年8月末時点でClaude Codeをどのように使っていたか、具体的な手順を中心に備忘録としてまとめたものです。まずは考え方を共有してから、具体の話をする構成となっています。

## 使い方の型を決める

Claude Codeを使用していると期待通りの挙動を引き出せず、延々と試行錯誤してしまうことがあります。しかし、実装タスクで集中するべきはツールの使い方ではなくタスク本来の内容であるため、なるべく使い方の型を決めてその通りに動かしています。

私の場合、型をつくるために以下のサイクルを回しました。

1. 普段の開発フローを整理して、改善できそうな箇所を見つける
2. Claude Codeを使うことでどのように改善できそうか調査をしたり、試作をつくって検証する
3. 実際の開発フローに組み込んでみる
4. 結果を振り返って、期待と実際の挙動にギャップが出てた場合は手順「1」に戻る

:::message
はじめから作業全体を効率化しようとせず、まずは部分的に取り入れてみることをお勧めします。使っているうちに得られる気づきが多いため、普段からClaude Codeの利用に慣れておくのがいいと思います。
:::

## インクリメンタルに進める

AI との協業を上手に進めるうえで「如何にしてコードの出力結果を安定させるか」を考える必要があります。常に十分な情報を提供できればよいのですが、人間側が正しく問題を理解できていない状況やコンテキストサイズの問題など、不確実性の高い状況は往々にして発生します。

そのための対策として、Claude Codeを**明確な作業スコープで可逆的に実行すること**が重要だと感じています。後述しますが、Gitのコミットと[difit](https://github.com/yoshiko-pg/difit)というローカルレビュー用のツールを使って、コミット単位でレビューを繰り返すことで開発を駆動します。

- コミットは細かく: プロンプトをコミットする。プロンプト単位でClaude Codeに実行結果をコミットさせる
- すべての作業を可逆にする: `git reset` でいつでも特定のバージョンに戻せる状態を維持する
- レビュー: 人間側が批判的に思考する機会をつくる
- 明確な指示: ファイルと行数を指定したうえで、明確な内容で作業を依頼する

## ターミナルを広く使う

個人的な感覚ですが、Claude Codeの入力欄は狭くて使いにくいです (よくエンターキーを誤爆します)。そこで、ターミナル上で動作する特徴を活かして、他のCLIツールで補える部分は慣れたツールで行うようにしています。基本的にはリポジトリ直下のMarkdownファイルにプロンプトを書いて、それを[スラッシュコマンド](https://docs.anthropic.com/en/docs/claude-code/slash-commands#custom-slash-commands)でClaude Codeに渡しています。

主に使用しているCLIツール:

- [Neovim](https://neovim.io/): ファイルの閲覧・編集など全般
- [Yazi](https://github.com/sxyazi/yazi): ファイルエクスプローラー
- [tmux](https://github.com/tmux/tmux): ターミナルのセッション管理や画面分割
- [ghq](https://github.com/x-motemen/ghq): リポジトリやGitブランチ間の移動

![](/images/acc1095edc0d6d/01d5acd4cd7a-20250829.png)
*Claude CodeとCLIツールの組み合わせ*

## ナレッジを蓄積する

Claude Codeを用いた開発の悩みとして、開発タスクを通じて得られるナレッジが自分自身に蓄積されにくいと感じています。そのための対策として、個人開発で作成したプロンプトやClaude Codeのアウトプット (レビューやタスクの内容をまとめたMarkdownファイル) は[Obsidian](https://obsidian.md/)で管理しています。

Obsidianはマルチデバイスの同期に対応している他、プレーンテキスト (Markdownファイル) で情報を扱えるため、ターミナル上のNeovimから簡単に編集できるといった利点を持っています。また、Claude Codeからも直接読み書きできるため、ナレッジを貯めて引き出すのに適したツールだと感じています。

![](/images/acc1095edc0d6d/62c640b36d71-20250831.png)
*Obsidianからナレッジを引き出す*

結果として、振り返りが効率化 (準備の時間を大幅に削減) されることで、自身で書いていないコードでも記憶の定着に繋げることができました。一方で技術の習得を目的とする場合、AIに考えさせた内容を写経するのはあまり効果を得られませんでした。

https://note.com/shotovim/n/n5833578984bf

## 現在の開発フロー

ここからは、Claude Codeを用いた具体的な開発フローを紹介します。本記事では扱わないため省略している箇所もありますが、大まかな手順は以下の通りです。

1. 作業ディレクトリの作成
2. 実装計画の作成
3. 実装計画の修正
4. 実装
5. 実装の修正
6. レビュー
7. 振り返りとナレッジ管理

### 作業ディレクトリの作成

[Anthropicのドキュメント](https://docs.anthropic.com/en/docs/claude-code/common-workflows)にもある通り、異なるブランチごとにClaude Codeのセッションを起動するために[git-worktree](https://git-scm.com/docs/git-worktree)を使用しています。今のところ、同時に複数の開発タスクを進めることはありませんが、以下のような場面でその利点を感じています。

- コードレビューの差込があった際、Claude Codeを別セッションで起動してレビューさせておきながら、自身はもともとのタスクを進行する
- 複数の実装方法がありどれがよいか迷っている際、Claude Codeを別セッションで起動してコンペを開催できる

しかし、ブランチ管理がやや煩雑になりルーチンワークも増えるため、開発フローに取り入れるには何らかの工夫が必要になると思います。私の場合はClaude CodeにCLIツールをつくらせましたが、[ccmanager](https://github.com/kbwo/ccmanager)などを使ってもよさそうです。

```bash
# `git worktree add` でディレクトリを作成して、その中に `.gitignore` に記載されたファイルやディレクトリをコピー
$ ccgw create fix-vitest-config
Preparing worktree (new branch 'fix-vitest-config')
HEAD is now at 74c13e8 feat: implement Vitest Browser Mode with Playwright for frontend testing
Found 6 files to copy based on .gitignore rules
Copied: apps/backend/.env
Copied: apps/frontend/next-env.d.ts
Copied: apps/frontend/node_modules/@redocly/openapi-core/tsconfig.tsbuildinfo
Copied: apps/frontend/node_modules/cliui/build/tsconfig.tsbuildinfo
Copied: apps/frontend/node_modules/is-arrayish/yarn-error.log
Copied: apps/frontend/tsconfig.tsbuildinfo
Worktree created successfully at: /Users/yuki/src/worktree/suibachi/fix-vitest-config
```

### 実装計画の作成 (`/design` コマンド)

作業ブランチができたら、実装計画のベースとなる要件定義書を用意します。私の場合は[Alfred](https://www.alfredapp.com/)にスニペットを登録しているので、それをもとにして`./tmp/context.md` に簡単な要件を書いています。

![](/images/acc1095edc0d6d/a3055930d84b-20250829.png)
*要件定義書のテンプレート*

ここで使用するのは、要件定義書から実装計画を作成する `/design` コマンドです。引数として任意のファイルパスを受け取れるため、`/design ./tmp/context.md` のようにして、前の手順で記載した内容をもとに、実装計画を `./tmp/plan.md` として出力します。

https://gist.github.com/31cefc0871c6403766d4c476f84308df

### 実装計画の修正 (`/revise` コマンド)

前の手順にて作成した実装計画ですが、一筆書きで満足いくものができることは滅多にありません。何度かレビューし、実装方針の調整やサンプルコードの追加でレールを敷いていきます。difitを起動してみます。

```bash
# HEADとmainの差分をすべて表示する
npx difit --mode inline HEAD main
```

![](/images/acc1095edc0d6d/f12acce57704-20250829.png)
*difitのレビュー画面*

ローカルサーバーが立ち上がり、ブラウザ上でGitHubに似たUIからレビューができます。実装計画に対するレビューコメントを入力し、「*Copy All Prompts*」ボタンを押すと以下のようなテキストがクリップボードにコピーされます。

```
apps/frontend/package.json:L33
test
=====
apps/frontend/package.json:L40
test2
```

このテキストを `./tmp/context.md` に貼り付け (必要に応じて編集) したうえで、実装計画を修正する `/revise` コマンドを実行します。あらかじめ定めた手順に従って実行されるため、`plan.md` のみを変更することや、difitからコピーしたテキストを入力として期待することなど、**特定の用途に特化した具体的な指示を効率的に与える**ことができます。これが開発フローをスラッシュコマンドという型で表現することの強みです。

満足がいくまで実装計画の修正を繰り返し、次の実装フェーズに入ります。

https://gist.github.com/af7a548e7691921a0261a17a298a7e19

### 実装 (`/implement` コマンド)

実装には `/implement` コマンドを使用します。実装計画の内容を理解して、受け入れ基準の達成を目標に実装してくれます。特に変わったことはありませんが、Claude Codeの書いたコードは不要なコメント (実装のアウトラインなど) が残ることが多いので、最後にこれらを削除させています。

https://gist.github.com/9e6a46609938cc45144615f5d59e0f7f

### 実装の修正

前のフェーズで実装の大枠はできるため、残りをコミットとレビューで駆動 (レビュー -> 実装 -> コミット) しながら完成に近づけます。実装の修正フェーズで主に使うコマンドとして、質問用の `/ask` コマンドと指示用の `/instruct` コマンドを紹介します。

#### 質問 (`/ask` コマンド)

まずは `/ask` コマンドですが、[planモード](https://x.com/_catwu/status/1955694117264261609)を模した回答特化のスラッシュコマンドです。ファイルを編集しないことに加え、Web検索の精度を上げるための工夫 ([Context7 MCP Server](https://github.com/upstash/context7)を使う/ハルシネーションの対策など) をコンテキストとして与えています。

基本的な使い方として、Claude Codeが出力したコードに対する疑問点を `./tmp/context.md` に記載してから実行します。得られた回答結果を踏まえて、後続の指示につなげます。

https://gist.github.com/ef22217d7499c128704bec164cfc5f33

#### 指示 (`/instruct` コマンド)

`/instruct` コマンドは受け取った指示を1件ずつ解釈して、[Explore, plan, code, commit](https://www.anthropic.com/engineering/claude-code-best-practices)をループで回すつくりになっています。指示の単位でコミットが分割されるため、レビュー時に変更を追いやすいです。

https://gist.github.com/9a9d1e62076ca27e21cb2a7c3dfa44ee

### レビュー (`/review` コマンド)

実装が完了したらPull Requestを作成し、レビューを実施します。新しくClaude Codeのセッションを立ち上げ `/review` コマンドを実行すると、複数の[サブエージェント](https://docs.anthropic.com/en/docs/claude-code/sub-agents)を利用したレビューが始まります。少し時間がかかるため、この間に自分でもセルフレビューを実施することが多いです。

![](/images/acc1095edc0d6d/93eb2a5e42c7-20250831.png)
*複数のサブエージェントが並列稼働する*

各サブエージェントはレビューのスコープを特定の技術領域に限定しており、より専門的なレビューが行えます。親タスクは変更された `git diff` の結果からどのサブエージェントを呼び出すか決定し、受け取った結果をまとめます。以下のリファクタリング用サブエージェントなどは、変更内容に関わらず必ず呼ばれるようにしました。

https://gist.github.com/d28609f27645358488982ea03c313ba1

レビュー結果として `./tmp/review.md` が出力されるため、これを参考にしながら機能を仕上げます。

https://gist.github.com/c97cc2cbbc6a64c97953011e615c0227

### 振り返り (`/recap` コマンド) とナレッジ管理

タスク完了後、`/recap` コマンドで実施した内容の振り返りを行います。結果はテンプレート化した内容に従って `./tmp/recap.md` として出力されるため、自分でも軽く覚書を書いておきます (後で思い出しやすくなる)。タスクで得られた副産物であるMarkdownファイル (`context.md`、`plan.md`、`review.md`、`recap.md`) はまとめてObsidianにコピーして、今後の学習や開発フローの改善に役立てます。

https://gist.github.com/a4b5ad9a8a2076a3ca9be9285ab0a106

## おわりに

今回はClaude Codeを用いた開発フローの一例を紹介しました。好みのツールや開発環境が人それぞれ違うように、エンジニアの数だけClaude Codeの適した使い方があるのだと思います。記事の中で取り上げたスラッシュコマンドやサブエージェントはGistで公開しているので、何かの参考になれば幸いです。