---
title: "NotionヘッドレスCMS構築方法"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp","Notion","GitHubActions"]
published: false
---

## 概要

Notion をヘッドレスCMS として活用し、記事を GitHub に自動で反映させる仕組みを構築する方法を紹介します。  
本記事では、Notion のデータベースを記事管理の中心に据え、GitHub Actions を活用して Notion の内容をマークダウン形式で GitHub リポジトリに同期させる手順を詳しく解説します。  

この方法を使うことで、Notion 上で記事を作成・編集するだけで、自動的に GitHub に反映されるようになります。  

## やりたいこと

1. Notionの勉強用データベースにその日の学習記録をつける。  
2. GitHubActionsでNotionデータベースから、その記事のマークダウンファイルを生成し、勉強用リポジトリにプッシュする。  

記事をNotionにまとめて、その活動記録をgithubの草として自動的に反映させたい。

似たようなことをしている記事があったのでまとめる。

しかし、どの記事でも具体的な方法が書かれていないので実現には至っていない。

上手くいっているリポジトリをフォークして色々やってみればいけるのでは？

## なぜこの記事を書いたのか？  

NotionをヘッドレスCMSとして使いたいと思って調べても、やり方だけが書かれていて、実際の構築方法が詳しく説明されているサイトがない！  
「それなら自分で詳しく書こう」と思ったのがこの記事を書いた理由です。  
本記事では、単なる手順の羅列ではなく、**なぜこの手順が必要なのか、どんな仕組みで動いているのかを含めて解説** します。  

![1000006240](https://github.com/user-attachments/assets/0309ead8-d6db-4ae4-846f-ce9bf8193e0d)

## そもそもNotionをヘッドレスCMSとして使うとは？  

普通のCMS（WordPressなど）と違い、NotionをヘッドレスCMSとして使う場合は以下のような仕組みになります。  

1. Notionのデータベースに記事を作成  
2. APIを使ってデータを取得  
3. 取得したデータをMarkdownに変換  
4. GitHub Actionsで定期的に取得＆デプロイ  

この流れを実現すると、**Notionで記事を管理しながら、自動的にGitHubにMarkdownとして保存する** ことができます。  

## 事前準備  

### 必要なツール  

- **Notion**（記事を管理するCMS代わり）  
- **Notion API**（Notionのデータを取得する）  
- **GitHub & GitHub Actions**（記事データをMarkdownに変換し管理する）  
- **C#**（APIと連携するプログラムを作成）  

## 実際に構築してみよう

### ① Notionで記事管理用データベースを作成  

1. Notionで新規データベースを作成  
2. 以下のプロパティを追加  
   - **Title**（記事のタイトル）  
   - **Slug**（記事のURL用スラッグ）  
   - **Content**（本文）  
   - **Published**（公開状態）  

### ② Notion APIの準備  

1. [Notion APIのインテグレーショントークン](https://programming-zero.net/notion-api-setting/)を取得  
2. [データベースID](https://note.com/amatyrain/n/nb9ebe31dfab7)を取得  
3. GitHubのシークレットに登録  

### ③ GitHub Actionsのセットアップ  

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

## C#でNotion APIと連携するコードを書く  

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

## まとめ  

- NotionをヘッドレスCMSとして活用するメリット  
- 実装の流れをおさらい  
- 今後の改善点や発展的な使い方  

### 🎯 こんな人に役立つ記事  

- **NotionをCMSとして使いたいが、構築方法が分からない人**  
- **GitHub ActionsでNotionの記事を自動管理したい人**  
- **実際に動くサンプルを試してみたい人**  

## 参考資料・リンク

[Notionでブログを書く | Yucchiy's Note](https://blog.yucchiy.com/2022/05/blogging-with-notion/)  
[C#でCustom GitHub Actionを書く | Yucchiy's Note](https://blog.yucchiy.com/2022/05/implement-custom-github-action-with-csharp/)  

[NotionヘッドレスCMS化記録 (3) GitHub Actionsと自動デプロイ | lacolaco's marginalia](https://blog.lacolaco.net/posts/notion-headless-cms-3/)

[結局Githubに学習履歴を統一した方が諸々良かった](https://zenn.dev/bun913/articles/study-history-on-github)  
