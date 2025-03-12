---
title: "Notionを使ったヘッドレスCMSの構築方法"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp","Notion","GitHubActions"]
published: true
---

## 概要

本記事では、NotionをヘッドレスCMSとして活用し、GitHub Actionsを用いてNotionのデータをMarkdownに変換し、自動でGitHubへ反映させる仕組みを紹介します。  

## 背景

学習記録をGitHubで管理するために、技術スタックごとにディレクトリを分けていましたが、複数技術が絡む場合の管理が煩雑でした。  
例えば、「C#でEFCoreを使ってデータベース操作をする」という記事を書いた場合、C#？ フレームワーク？ データベース？ どのディレクトリに配置するのが最適なのか決めかねる、といった具合です。  
そこで、**Notionのデータベースを活用し、タグで技術スタックを管理する方法**を考えました。  

しかし、**学習記録はNotionに蓄積されるものの、GitHubには反映されない**ため、せっかくの学習成果が形として残らないのがもったいないと感じ始めました。  

「Notionに記録するだけで自動的にGitHubに同期される仕組み」があれば理想的だと考えました。  

## 理想の運用フロー

1. Notionのデータベースに学習記録を追加する  
2. 学習記録が自動でMarkdown化され、GitHubのリポジトリにプッシュされる  

このような仕組みを実現する方法を調べたところ、GitHub Actionsを活用すれば可能であることが分かりました。  

## なぜこの記事を書いたのか？

GitHub Actions の yml設定やスクリプトを紹介する記事は多くありましたが、ゼロからシステムを構築する具体的な方法を解説したサイトは見つかりませんでした。  
そのため、本記事では GitHub Actions の知識がない状態から構築した経験をもとに、ゼロベースからの構築方法をまとめることにしました。  

## ヘッドレスCMS

この仕組みは「**ヘッドレスCMS**」に分類される様です。  
既存の事例を参考にしつつ、本記事でも同様の用語を使用します。  

参考記事:  
<!-- NotionヘッドレスCMS化記録 (3) GitHub Actionsと自動デプロイ | lacolaco's marginalia -->
https://blog.lacolaco.net/posts/notion-headless-cms-3/  

### ヘッドレスCMSとは？

AIに聞いてみたところ、以下のような回答をしてくれました。  

>ヘッドレスCMS（Headless CMS）とは、コンテンツの管理（バックエンド）と表示（フロントエンド）を分離したコンテンツ管理システムのことです。通常のCMS（例えば WordPress など）は、管理画面でコンテンツを作成・編集し、同じシステム内でウェブサイトとして表示します。一方、ヘッドレスCMSはコンテンツ管理のみを担当し、APIを通じてデータを外部のWebサイトやアプリに提供します。
>
>「本システムでは、Notion をコンテンツ管理のバックエンドとして利用し、GitHub Actions を通じてデータを変換・反映するため、ヘッドレスCMS の概念に当てはまります。」  

だそうです。  

## 前提・環境

- OS: Windows 11 HOME 24H2  
- 言語: C#
- フレームワーク: .NET 8
- ツール: VSCode, GitHub Actions

実装にはC#を採用しました。理由は以下の2点です。  

- 最も得意な言語であるため、スムーズに開発できる  
- 既存のC#製の変換スクリプトがあり、それを活用できるため  

参考リポジトリ:  
https://github.com/yucchiy/notion-to-markdown  

正直、このリポジトリがあったからこそ実装出来たようなものでした。  
本サンプルは、こちらで公開されているC#プロジェクトを `.Net8` に対応させたのと、細かい部分を私なりにアレンジした物となります。  

## 実装・解説

実装は以下の5ステップで進めます。  

1. **Notionの設定**  
   - **1-1.** Notionデータベースを作成 & プロパティの設定  
   - **1-2.** NotionAPI利用のためのIntegration Tokenの取得  
   - **1-3.** Notion Database ID の取得  
2. **GitHubリポジトリの設定**  
   - **2-1.** リポジトリの作成とプロジェクト構成  
   - **2-2.** リポジトリのシークレットに登録  
   - **2-3.** リポジトリのpermissionsの設定  
3. C#プロジェクトの作成  
4. GitHub Actionsのワークフローファイルの作成  
5. GitHubにPush  

### 1. Notionの設定

#### 1-1. Notionデータベースを作成 & プロパティの設定

まずはNotionで記事管理用のデータベースを作成します。  

フルページでデータベースを作成してください。データベース名は何でも構いません。  
※今回はサンプルとして `notion-headless-cms-sample-database` としました。  

次にプロパティを設定していきます。  
以下のプロパティを追加してください。GitHub連携の際に必ず必要となります。  

| プロパティ | プロパティの種類 | 用途 |
|---|---|---|
| **Title** | タイトル | 記事のタイトル名です。ヘッダー情報になります。 |
| **Slug** | テキスト | GitHubで記事のディレクトリ名になります。指定しない場合はタイトルがディレクトリ名となります。 |
| **Type** | セレクト | Zennのように記事が`Idea`か`Tech`なのかを示します。ヘッダー情報になります。 |
| **Tags** | マルチセレクト | 技術スタック等を選択するものです。ヘッダー情報になります。 |
| **Description** | テキスト | 記事の説明です。 |
| **RequestPublishing** | チェックボックス | 記事公開フラグです。チェックがついている記事が公開の対象となります。 |
| **PublishedAt** | 作成日時 | 記事を作成した日時です。ヘッダー情報になります。 |
| **_SystemCrawledAt** | 最終更新日時 | プログラムからデータベースを操作するため、更新日時を反映させる為に必要となります。 |

![alt text](/images/notion-headless-cms-sample/notion-database.png)  

サンプルNotionページ:  
https://periodic-cheese-dfa.notion.site/19e4a91b2eac80928ca1ca4acaa42933?v=19e4a91b2eac817b8c51000cd6f66e99  

#### 1-2. NotionAPI利用のためのIntegration Tokenの取得

こちらの記事を参考に**Integration Token**なるものを取得してください。  
<!-- Notion APIのインテグレーショントークン -->
https://programming-zero.net/notion-api-setting/  

このトークンは、次の工程(2-2. リポジトリのシークレットに登録)で必要となるのでメモしておいてください。  

#### 1-3. Notion Database ID の取得

**Notion Database ID** はこちらの記事を参考に取得してください。  
<!--【Notion】データベースIDを確認しよう｜あまてぃ  -->
https://note.com/amatyrain/n/nb9ebe31dfab7  

こちらは簡単なのでそのまま引用させて頂きます。  
>③URLからデータベースIDを確認
>
>URLは「`https://www.notion.so/XXXXXXXX?v=YYYYYYYY`」といった形式となっていますが、その「`XXXXXXXX`」の部分がデータベースIDとなります  

こちらのIDも、次の工程(2-2. リポジトリのシークレットに登録)で必要となるのでメモしておいてください。  

### 2. GitHubリポジトリの設定  

#### 2-1. リポジトリの作成とプロジェクト構成

新規でリポジトリを作成してください。リポジトリ名は何でも構いません。  
※今回はサンプルとして `notion-headless-cms-sample` としました。  

ここでプロジェクト構成について確認しておきます。  
今後の作業において、以下のようなプロジェクト構成でサンプルを作成していきます。  

``` txt
notion-headless-cms-sample/
├── .github/
│   └── workflows/
│       └── workflow.yml        # GitHub Actionsのワークフローファイル
├── src/
│   ├── .gitignore              # ignore file
│   ├── Program.cs              # メインプログラム
│   └── NotionToMarkdown.csproj # プロジェクトファイル
└── articles/                   # 記事ディレクトリ
    └── yyyy/
        ├── mm/
        │   ├── Hoge.md
        │   └── Fuga.md
        └── mm/
            └── Piyo.md
```

#### 2-2. リポジトリのシークレットに登録

こちらの記事を参考にしながら設定すると分かりやすいと思います。
<!-- GitHub Actions のシークレット情報と変数の設定方法 #GitHubActions - Qiita -->
https://qiita.com/mkin/items/75a4928a1fafe5eacd17  

- **リポジトリのSettingsタブ** → **Secrets and variablesのアコーディオン内のActions** → **New repository secret** ボタンを押下  

Notionの設定時にメモしておいた「**Integration Token**」と「**Notion Database ID**」を登録してください。  
メールアドレスとユーザー名に関しては、GitHub Actionsを実行した時に草を生やすために必要です。  

| 変数名 | 格納する値 |
|---|---|
| **NOTION_AUTH_TOKEN** | 1-2.でメモしておいた **Integration Token** |
| **NOTION_DATABASE_ID** | 1-3.でメモしておいた **Notion Database ID** |
| **USER_EMAIL** | githubに登録しているメールアドレス |
| **USER_NAME** | githubのユーザー名 |

#### 2-3. リポジトリのpermissionsの設定

次にリポジトリの`Workflow permissions`を設定していきます。  

- **リポジトリのSettingsタブ** → **Actionsのアコーディオン内のGeneral** → **Workflow permissions** のラジオボタンを確認 → 「**Read and Write permissions**」 に変更

初期状態は 「**Read repository contents and packages permissions**」 となっていると思いますが、これを 「**Read and Write permissions**」 に変更してください。  
ここを変更しておかないと、GitHub Actionsを実行した時に`403`エラーとなってしまいます。  

![alt text](/images/notion-headless-cms-sample/github-actions-permission-error.png)

- **Read and Write permissions(読み取りと書き込みの権限)**  
ワークフローには、すべてのスコープに対してリポジトリへの読み取りと書き込みの権限があります。  
- **Read repository contents and packages permissions(リポジトリのコンテンツとパッケージの読み取り権限)**  
ワークフローは、リポジトリのコンテンツとパッケージのスコープに対してのみ読み取り権限を持ちます。  

### 3. C#プロジェクトの作成  

GitHub Actionsで実行する処理を作成していきます。  

次の2つの環境での作業を想定して進めます。  

- `git clone`してローカルでVSCodeで作業する  
- GitHubのCodespacesで作業する  

また、作業方法としては次の2つを想定しています。  

1. 各ファイルを手動で作成&コピペで進める方法  
2. dotnetコマンドで進める方法  

dotnetに詳しい方であれば、2の方法を取れると思いますが、とりあえず実現するだけなら手動でファイルを作成してコピペでいけます。  

どの方法にせよ、次の3ファイルを用意してコードをコピペすればOKです。  

- `NotionToMarkdown.csproj (プロジェクトファイル)`
- `Program.cs (プログラムファイル)`
- `.gitignore (ignoreファイル)`

``` txt
notion-headless-cms-sample/
├── src/
│   ├── .gitignore
│   ├── Program.cs
│   └── NotionToMarkdown.csproj
```

#### 1. 各ファイルを手動で作成&コピペで進める方法  

項目名の通りです。  
`src`ディレクトリを作成して、その中にファイルを作成して行きます。  

`NotionToMarkdown.csproj`という名前でファイルを作成して、次のコードをコピペしてください。  
https://github.com/rendya2501/notion-headless-cms-sample/blob/main/src/NotionToMarkdown.csproj  

`Program.cs`という名前でファイルを作成して、次のコードをコピペしてください。  
https://github.com/rendya2501/notion-headless-cms-sample/blob/main/src/Program.cs  

`.gitignore`という名前でファイルを作成して、次のコードをコピペしてください。  
https://github.com/rendya2501/notion-headless-cms-sample/blob/main/src/.gitignore  

<!-- https://github.com/rendya2501/notion-headless-cms-sample/tree/main/src -->
手動で進める方法は以上となります。  

#### 2. dotnet コマンドで進める方法

`dotnet` コマンドで作業を行う手順も載せておきます。  
※ローカルで作業する場合、`.NET SDK`をインストールしておいてください。  

- **プロジェクトの作成**  

  ``` bash
  dotnet new console -f net8.0 -o src -n NotionToMarkdown
  ```

  バージョンは `.net8` で、プロジェクトファイルの出力先は`src`ディレクトリ、プロジェクト名は`NotionToMarkdown`でコンソールプロジェクトを作成する。  

- **依存関係の追加**  

  ``` bash
  cd src
  dotnet add package Notion.Net --version 4.2.0
  dotnet add package Scriban --version 5.12.1
  ```

  - **Notion.Net**  
    Notion APIを利用するためのC#ライブラリで、Notionのデータベースやページにアクセスして操作することができます。  
  - **Scriban**  
    高速で柔軟なテンプレートエンジンで、テンプレートを使って文字列を生成するのに役立ちます。  

- **ignoreファイルの作成**  

  ``` bash
  dotnet new gitignore
  ```

  dotnetのビルド生成フォルダ `obj`、`bin`をgitの追跡対象から除外するために必要です。  

- **Program.csの内容をリポジトリからコピペ**

  同じくコードに関しては、次のコードをコピペしてください。  

https://github.com/rendya2501/notion-headless-cms-sample/blob/main/src/Program.cs  

### 4. GitHub Actionsのワークフローファイルの作成  

GitHub Actionsを実行するためのワークフローファイルを作成していきます。  
`.github/workflows`ディレクトリを作成し、その中に`workflow.yml`ファイルを作成してください。  

``` txt
notion-headless-cms-sample/
├── .github/
│   └── workflows/
│       └── workflow.yml
```

ファイルの内容は次の通りです。  
`workflow.yml`ファイルにコピペしてください。  
https://github.com/rendya2501/notion-headless-cms-sample/blob/main/.github/workflows/workflow.yml

### 5. GitHubにPush

最後に、ここまでの作業内容をリポジトリにpushしてください。  

以上で作業は終わりです。
お疲れさまでした!  

## デモ

`Test1(Fuga)`、`Test2(Piyo)`、`Test4`を公開対象とします。  

![alt text](/images/notion-headless-cms-sample/selected.png)

`Test1`の記事の内容は以下の通りです。  

![alt text](/images/notion-headless-cms-sample/article.png)

GitHub Actionsを実行します。  

![alt text](/images/notion-headless-cms-sample/run-workflow.png)

GitHub Actionsが成功するとNotionの記事がマークダウンとして生成され、GitHubに草が生えます。  
画像の記事は`Test1(Fuga)`の物である事が分かると思います。  
また、ディレクトリツリーより`Test2(Piyo)`、`Test4`の記事も生成されている事が確認出来ると思います。  

![alt text](/images/notion-headless-cms-sample/github-article-demo.png)
![alt text](/images/notion-headless-cms-sample/selected.png)  

こちらのサンプルではこれが精一杯ですが、これをベースに自分なりにアレンジしてみるのも良いかもしれません。  

## 参考サイト

<!-- Notionでブログを書く | Yucchiy's Note -->
https://blog.yucchiy.com/2022/05/blogging-with-notion/  

<!-- C#でCustom GitHub Actionを書く | Yucchiy's Note -->
https://blog.yucchiy.com/2022/05/implement-custom-github-action-with-csharp/

<!-- yucchiy/notion-to-markdown -->
https://github.com/yucchiy/notion-to-markdown  

<!-- NotionヘッドレスCMS化記録 (3) GitHub Actionsと自動デプロイ | lacolaco's marginalia -->
https://blog.lacolaco.net/posts/notion-headless-cms-3/  

<!-- 結局Githubに学習履歴を統一した方が諸々良かった -->
https://zenn.dev/bun913/articles/study-history-on-github   

## リポジトリ

この記事で作成した成果物を置いています。  
参考にしてください。  

https://github.com/rendya2501/notion-headless-cms-sample

https://periodic-cheese-dfa.notion.site/19e4a91b2eac80928ca1ca4acaa42933?v=19e4a91b2eac817b8c51000cd6f66e99  

