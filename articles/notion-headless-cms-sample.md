---
title: "NotionヘッドレスCMS構築方法"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp","Notion","GitHubActions"]
published: false
---

## 概要

本記事では、Notion のデータベースを記事管理の中心に据え、GitHub Actions を活用して Notion の内容をマークダウン形式で出力し GitHub リポジトリに同期させる機能の実現方法を解説します。  

この方法を使うことで、Notion 上で記事を作成・編集するだけで、自動的に GitHub に反映されるようになります。  

## やりたかったこと

1. Notionの勉強用データベースにその日の学習記録をつける  
2. GitHubActionsでNotionデータベースから、その記事のマークダウンファイルを生成し、勉強用リポジトリにプッシュする  

記事をNotionにまとめて、その活動記録をgithubの草として自動的に反映させたい。  

## ヘッドレスCMS

このようなシステムを `ヘッドレスCMS` と言うらしいです。  
先人たちがそのように呼称していたので、自分も同じように使わせて貰います。  
[NotionヘッドレスCMS化記録 (3) GitHub Actionsと自動デプロイ | lacolaco's marginalia](https://blog.lacolaco.net/posts/notion-headless-cms-3/)  
[Notionでブログを書く | Yucchiy's Note](https://blog.yucchiy.com/2022/05/blogging-with-notion/)  

※参考・リンクにもまとめています。  

### 「そもそもヘッドレスCMSとは何ぞや？」

AIの回答をそのまま引用しておきます。  

>ヘッドレスCMS（Headless CMS）とは、コンテンツの管理（バックエンド）と表示（フロントエンド）を分離したコンテンツ管理システムのことです。通常のCMS（例えば WordPress など）は、管理画面でコンテンツを作成・編集し、同じシステム内でウェブサイトとして表示します。一方、ヘッドレスCMSはコンテンツ管理のみを担当し、APIを通じてデータを外部のWebサイトやアプリに提供します。

## なぜこの記事を書いたのか？  

NotionをヘッドレスCMSとして運用させる記事はあれど、具体的な構築方法まで解説してくれているサイトがない。  
参考サイトやAIを駆使して試行錯誤した結果、なんとか実現する事が出来たので、そこまでの過程をまとめておきたいと思ったので書きました。(技術記事のネタにもなるし)  

## 前提

このシステムを構築するにあたり、筆者はGitHubActionsなんて使った事も触った事もありませんでした。  
なので、いくらシステムを構築する事が出来たとはいえ、拙い部分があるかもしれませんので、そこはご了承ください。  
また、一番得意な言語がC#だったので、C#中心の調査をして実装しています。  
似たような境遇の方の参考になれば幸いです。  

<!-- 
- **Notion**（記事を管理するCMS代わり）  
- **Notion API**（Notionのデータを取得する）  
- **GitHub & GitHub Actions**（記事データをMarkdownに変換し管理する）  
- **C#**（APIと連携するプログラムを作成）   
-->

## 開発環境  

- Win11 24H2  
- .Net 8  
- VSCode or VisualStudio2022  

## 実装・解説

以下の手順で実装していきます。  

1. Notionの設定  
   1. Notionデータベースを作成
   2. GitHubとの連携に必要なプロパティの作成
   3. NotionAPI利用のためのインテグレーショントークンの取得
   4. Notion Database ID の取得
2. GitHubリポジトリの設定
   1. GitHubリポジトリを作成
   2. リポジトリのシークレットに登録
   3. GitHub Actionsのための設定
3. C#プロジェクトの作成  
4. GitHubActionsの設定  

### 1-1. Notionで記事管理用データベースを作成  

1. Notionで新規データベースを作成  

### 1-2. GitHubとの連携に必要なプロパティの作成

以下のプロパティを追加  

- **Title**（記事のタイトル）  
- **Slug**（記事のURL用スラッグ）  
- **Content**（本文）  
- **Published**（公開状態）  

### 1-3. NotionAPI利用のためのインテグレーショントークンの取得

### 1-4. Notion Database ID の取得

1. [Notion APIのインテグレーショントークン](https://programming-zero.net/notion-api-setting/)を取得  
2. [データベースID](https://note.com/amatyrain/n/nb9ebe31dfab7)を取得  
3. GitHubのシークレットに登録  

### 3. GitHub Actionsのセットアップ  

GitHub Actionsを使い、定期的にNotionからMarkdownを取得するように設定します。  
以下のYAMLファイルを作成（`.github/workflows/workflow.yaml`）  

```yaml
name: Notion to Markdown
on:
  schedule:
    - cron: '0 0 * * *'  # 毎日0時に実行
  workflow_dispatch:

jobs:
  fetch-notion:
    runs-on: ubuntu-latest
    steps:
      - name: リポジトリのチェックアウト
        uses: actions/checkout@v3

      - name: Notionデータの取得
        run: |
          echo "Fetching data from Notion..."
          # C#スクリプトを実行
```

### 4. C#でNotion APIと連携するコードを書く  

`Program.cs` に記述するコードの概要を解説しながら書く（例）  

```csharp
// Notion API からデータを取得し Markdown に変換する C# コード
using System;
using System.Net.Http;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        var client = new HttpClient();
        var notionApiUrl = "https://api.notion.com/v1/databases/{DATABASE_ID}/query";
        client.DefaultRequestHeaders.Add("Authorization", "Bearer {YOUR_NOTION_TOKEN}");
        client.DefaultRequestHeaders.Add("Notion-Version", "2022-06-28");

        var response = await client.PostAsync(notionApiUrl, null);
        var content = await response.Content.ReadAsStringAsync();

        Console.WriteLine(content);
    }
}
```

**なぜこのコードが必要なのか、どの部分が重要なのかを解説** しながら進める。  

## 参考・リンク

[Notionでブログを書く | Yucchiy's Note](https://blog.yucchiy.com/2022/05/blogging-with-notion/)  
[C#でCustom GitHub Actionを書く | Yucchiy's Note](https://blog.yucchiy.com/2022/05/implement-custom-github-action-with-csharp/)  

[NotionヘッドレスCMS化記録 (3) GitHub Actionsと自動デプロイ | lacolaco's marginalia](https://blog.lacolaco.net/posts/notion-headless-cms-3/)

[結局Githubに学習履歴を統一した方が諸々良かった](https://zenn.dev/bun913/articles/study-history-on-github)  

<!--  -->

## メモ

Notion をヘッドレスCMS として活用し、記事を GitHub に自動で反映させる仕組みを構築する方法を紹介します。  

![1000006240](https://github.com/user-attachments/assets/0309ead8-d6db-4ae4-846f-ce9bf8193e0d)

## そもそもNotionをヘッドレスCMSとして使うとは？  

普通のCMS（WordPressなど）と違い、NotionをヘッドレスCMSとして使う場合は以下のような仕組みになります。  

1. Notionのデータベースに記事を作成  
2. APIを使ってデータを取得  
3. 取得したデータをMarkdownに変換  
4. GitHub Actionsで定期的に取得＆デプロイ  

この流れを実現すると、**Notionで記事を管理しながら、自動的にGitHubにMarkdownとして保存する** ことができます。  

## まとめ  

- NotionをヘッドレスCMSとして活用するメリット  
- 実装の流れをおさらい  
- 今後の改善点や発展的な使い方  

### 🎯 こんな人に役立つ記事  

- **NotionをCMSとして使いたいが、構築方法が分からない人**  
- **GitHub ActionsでNotionの記事を自動管理したい人**  
- **実際に動くサンプルを試してみたい人**  
