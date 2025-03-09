---
title: "Notionを使ったヘッドレスCMSの構築方法"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp","Notion","GitHubActions"]
published: false
---

## 概要

この記事では、NotionをヘッドレスCMSとして活用し、GitHub Actionsを用いてNotionのデータをMarkdown化し、自動でGitHubに反映させる仕組みを紹介します。  

## 背景

学習記録をGitHubで管理するために、技術スタックごとにディレクトリを分けていましたが、複数技術が絡む場合の管理が煩雑でした。  
例えば、「C#でEFCoreを使ってデータベース操作をする」という記事を書いた場合、C#？ フレームワーク？ データベース？ どのディレクトリが最適なのか決めかねる、といった具合です。  
そこで、**Notionのデータベースを活用し、タグで技術スタックを管理する方法**を考えました。  

しかし、**学習記録はNotionに蓄積されるものの、GitHubには反映されない**ため、せっかくの学習成果が形として残らないのがもったいないと感じ始めました。  
「Notionに記録するだけで自動的にGitHubに同期される仕組み」があれば理想的だと考えました。  

## 理想の運用フロー

1. Notionのデータベースに学習記録を追加  
2. 記録が自動でMarkdown化され、GitHubのリポジトリにプッシュされる  

このような仕組みを実現する方法を調べたところ、GitHub Actionsを活用すれば可能であることが分かりました。  

## なぜこの記事を書いたのか？

GitHub Actions の yml設定やスクリプトを紹介する記事は多くありましたが、ゼロからシステムを構築する具体的な方法を解説したサイトは見つかりませんでした。  
そのため、本記事では GitHub Actions の知識がない状態から構築した経験をもとに、ゼロベースからの構築方法をまとめることにしました。  

## ヘッドレスCMS

このようなシステムを `ヘッドレスCMS` と言うらしいです。  
先人たちがそのように紹介していたので、自分も同じように使わせて貰います。  

参考記事:  
<!-- NotionヘッドレスCMS化記録 (3) GitHub Actionsと自動デプロイ | lacolaco's marginalia -->
https://blog.lacolaco.net/posts/notion-headless-cms-3/  

### ヘッドレスCMSとは？

AIの回答をそのまま引用しておきます。  

>ヘッドレスCMS（Headless CMS）とは、コンテンツの管理（バックエンド）と表示（フロントエンド）を分離したコンテンツ管理システムのことです。通常のCMS（例えば WordPress など）は、管理画面でコンテンツを作成・編集し、同じシステム内でウェブサイトとして表示します。一方、ヘッドレスCMSはコンテンツ管理のみを担当し、APIを通じてデータを外部のWebサイトやアプリに提供します。
>
>「本システムでは、Notion をコンテンツ管理のバックエンドとして利用し、GitHub Actions を通じてデータを変換・反映するため、ヘッドレスCMS の概念に当てはまります。」  

だそうです。  

## 前提・環境

- OS: Windows 11 HOME 24H2  
- 言語: C#
- フレームワーク: .NET 8
- ツール: VSCode, GitHub Actions

実装言語はC#です。理由は2つあります。  

1. 自分が一番得意な言語だから  
2. C#で実装されている変換スクリプトがあったから  

https://github.com/yucchiy/notion-to-markdown  

正直、このリポジトリが無かったら実装出来ていませんでした。  
本サンプルは、こちらで公開されているC#コードを `.Net8` に対応させたのと、細かい部分を自分なりにアレンジした物となります。  

## 実装・解説

以下の手順で実装していきます。  

1. Notionの設定  
   1. Notionデータベースを作成&プロパティの設定  
   2. NotionAPI利用のためのインテグレーショントークンの取得
   3. Notion Database ID の取得
2. GitHubリポジトリの設定
   1. リポジトリの作成とプロジェクト構成
   2. リポジトリのシークレットに登録
   3. リポジトリのpermissionsの設定
3. C#プロジェクトの作成  
4. GitHub Actionsのワークフローファイルの作成  

### 1. Notionの設定

#### 1-1. Notionデータベースを作成 & プロパティの設定

まずはNotionで記事管理用のデータベースを作成します。  

フルページでデータベースを作成してください。  
データベース名は何でも構いません。  
※自分はサンプルとして `notion-headless-cms-sample-database` としました。  

次にプロパティを設定していきます。  
以下のプロパティを追加してください。  
GitHub連携の際に必ず必要となります。  

| プロパティ | プロパティの種類 | 用途 |
|---|---|---|
| **Title** | タイトル | 記事本体のタイトル名です。ヘッダー情報に含めます。 |
| **Slug** | テキスト | GitHubで記事のディレクトリ名になります。指定しない場合はタイトルがディレクトリ名となります。 |
| **PublishedAt** | 作成日時 | 記事を作成した日時です。ヘッダー情報に含めます。 |
| **Type** | セレクト | Zennのように記事が`Idea`か`Tech`なのかを示します。ヘッダー情報に含めます。 |
| **Tags** | マルチセレクト | 技術スタック等を選択するものです。ヘッダー情報に含めます。 |
| **RequestPublishing** | チェックボックス | 記事公開フラグです。これにチェックがついている記事が公開の対象となります。 |
| **_SystemCrawledAt** | 最終更新日時 | プログラムからデータベースを操作するため、更新日時を反映させる為に必要となります。 |

![alt text](/images/notion-headless-cms-sample/notion-database.png)

#### 1-2. NotionAPI利用のためのインテグレーショントークンの取得

こちらの記事を参考に`インテグレーショントークン`なるものを取得してください。  
<!-- Notion APIのインテグレーショントークン -->
https://programming-zero.net/notion-api-setting/  
`GitHub Actions` の `Seacrets` で使うのでメモしておいてください。  

#### 1-3. Notion Database ID の取得

`Notion Database ID` はこちらの記事を参考に取得してください。  
<!--【Notion】データベースIDを確認しよう｜あまてぃ  -->
https://note.com/amatyrain/n/nb9ebe31dfab7  

こちらは簡単なのでそのまま引用させて頂きます。  
>③URLからデータベースIDを確認
>
>URLは「`https://www.notion.so/XXXXXXXX?v=YYYYYYYY`」といった形式となっていますが、その「`XXXXXXXX`」の部分がデータベースIDとなります  

こちらも `GitHub Actions` の `Seacrets` で使うのでメモしておいてください。  

### 2. GitHubリポジトリの設定  

#### 2-1. リポジトリの作成とプロジェクト構成

新規でリポジトリを作成してください。リポジトリ名は何でも構いません。  
※自分はサンプルとして `notion-headless-cms-sample` としました。  

ここでプロジェクト構成について確認しておきます。  
今後の作業において、以下のようなプロジェクト構成でサンプルを作成していきます。  

後、このタイミングで `articles` ディレクトリを作成して `.keep` ファイルを作成しておいてください。  
`articles` ディレクトリを追跡対象とする為にダミーファイルを作成します。  
※`articles` ディレクトリが存在しないとマークダウンファイルが生成されません…

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
    ├── yyyy/
    |   ├── mm/
    |   │   ├── Hoge.md
    |   │   └── Fuga.md
    |   └── mm/
    |       └── Piyo.md
    └── .keep # articlesディレクトリを追跡対象とするためのダミーファイル
```

#### 2-2. リポジトリのシークレットに登録

こちらの記事を参考にしながら設定すると分かりやすいと思います。
<!-- GitHub Actions のシークレット情報と変数の設定方法 #GitHubActions - Qiita -->
https://qiita.com/mkin/items/75a4928a1fafe5eacd17  

- `リポジトリのSettingsタブ`
- `Secrets and variablesのアコーディオン内のActions`  
- `New repository secret` ボタンを押下  

Notionの設定時にメモしておいた`インテグレーショントークン`と`Notion Database ID`を登録してください。  
`メールアドレス`と`ユーザー名`に関してはGitHub Actionsを実行した時に草を生やすために必要です。  

| 変数名 | 格納する値 |
|---|---|
| **NOTION_AUTH_TOKEN** | `1-2.` で取得した **Notion APIのインテグレーショントークン** |
| **NOTION_DATABASE_ID** | `1-3.` で取得した **Notion Database ID** |
| **USER_EMAIL** | githubに登録しているメールアドレス |
| **USER_NAME** | githubのユーザー名 |

#### 2-3. リポジトリのpermissionsの設定

次にリポジトリの`Workflow permissions`を設定していきます。  

- `リポジトリのSettingsタブ`  
- `Actionsのアコーディオン内のGeneral`  
- `Workflow permissions` のラジオボタンを確認  
- `Read and Write permissions` に変更

初期状態は `Read repository contents and packages permissions` となっていると思いますが、これを `Read and Write permissions` に変更してください。  
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
- GitHubの`Codespaces`で作業する  

また、作業方法としては次の2つを想定しています。  

- 完全に手動でファイルを作成して作業する方法  
- dotnetコマンドでプロジェクトを作成して作業する方法  

どちらの方法にせよ、次の3ファイルを用意して、内容はサンプルリポジトリからコピペすればOKです。  

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

各ファイルの内容はサンプルリポジトリからコピペしてください。  
https://github.com/rendya2501/notion-headless-cms-sample/tree/main/src/NotionToMarkdown.csproj  
https://github.com/rendya2501/notion-headless-cms-sample/tree/main/src/Program.cs  
https://github.com/rendya2501/notion-headless-cms-sample/tree/main/src/.gitignore  

#### dotnetコマンドで進める方法

一応、`dotnet` コマンドで作業を進める手順も載せておきます。  

- **プロジェクトの作成**  

  ``` bash
  dotnet new console -f net8.0 -o src -n NotionToMarkdown
  ```

  バージョンは.net8で、プロジェクトファイルの出力先は`src`ディレクトリ、プロジェクト名は`NotionToMarkdown`のコンソールプロジェクトを作成する。  

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

  飛ばしても良い作業ですが、GitHubに上げるなら作っておいた方が良いです。  
  `obj`や`bin`フォルダをgitの追跡対象から除外します。  

- **Program.csの内容をリポジトリからコピペ**

https://github.com/rendya2501/notion-headless-cms-sample/blob/main/src/Program.cs

### 4. GitHub Actionsのワークフローファイルの作成  

GitHub Actionsを実行するためのワークフローファイル(`yml`)を作成していきます。  
`.github/workflows`ディレクトリを作成し、その中に`workflow.yml`ファイルを作成してください。  

``` txt
notion-headless-cms-sample/
├── .github/
│   └── workflows/
│       └── workflow.yml
```

ファイルの内容は次の通りです。  
`workflow.yml`ファイルにコピペしてください。  

:::details workflow.yml

```yml
name: Notion to Markdown Workflow

on:
  workflow_dispatch:
  # 23:30 に実行する(9時間の時差を考慮)
  schedule:
    - cron: '30 14 * * *'

jobs:
  notion_to_markdown:
    runs-on: ubuntu-latest

    steps:
      # リポジトリをチェックアウトするステップ
      - name: Checkout repository
        uses: actions/checkout@v4

      # .NETをセットアップするステップ
      - name: Set up .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0'

      # 依存関係を復元するステップ
      - name: Restore dependencies
        run: dotnet restore src/NotionToMarkdown.csproj

      # プロジェクトをビルドするステップ
      - name: Build
        run: dotnet build src/NotionToMarkdown.csproj --configuration Release --no-restore

      # 実行ファイルを作成するステップ
      # プロジェクトをビルドし、実行可能な形式に変換して出力フォルダーに配置します
      - name: Publish
        run: dotnet publish src/NotionToMarkdown.csproj --configuration Release --output ./out --no-build

      # NotionからMarkdownに変換するプログラムを実行するステップ
      - name: Run Notion to Markdown
        run: |
          dotnet ./out/NotionToMarkdown.dll \
            ${{ secrets.NOTION_AUTH_TOKEN }} \
            ${{ secrets.NOTION_DATABASE_ID }} \
            "articles/{{publish|date.to_string('%Y/%m')}}/{{slug}}"

      # エクスポートされたファイル数を確認するステップ
      # ファイル数が0の場合は、処理終了
      - name: Check for exported files
        if: env.EXPORTED_COUNT == '0'
        run: exit 0

      # エクスポートされたMarkdownファイルをリポジトリに反映するステップ
      - name: Commit exported markdown files
        run: |
          git config --global user.email "${{ secrets.USER_EMAIL }}"
          git config --global user.name "${{ secrets.USER_NAME }}"
          git add articles/
          git commit -m "Import files from notion database" || exit 0
          git push
```

:::

## デモ

まずNotionのデータベースがありまして、  

![alt text](/images/notion-headless-cms-sample/notion-database.png)

こんな感じの記事を書いたとします。  

![alt text](/images/notion-headless-cms-sample/notion-article1.png)
![alt text](/images/notion-headless-cms-sample/notion-article2.png)

GitHub Actionsを実行しまして、

![alt text](/images/notion-headless-cms-sample/github-actions-demo.png)

それが成功するとNotionの記事がマークダウンとして生成され、GitHubに草が生えます。  

![alt text](/images/notion-headless-cms-sample/github-article-demo.png)

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
