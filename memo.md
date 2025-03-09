# メモ

## zenn-cli

※コマンドはzenn-docsのルートで実行すること  

記事の作成コマンド  
`npx zenn new:article`  

記事のプレビューコマンド  
`npx zenn preview`  

[GitHubリポジトリでZennのコンテンツを管理する](https://zenn.dev/zenn/articles/connect-to-github)  
[Zenn CLIをインストールする](https://zenn.dev/zenn/articles/install-zenn-cli)  
[Zenn CLIで記事・本を管理する方法](https://zenn.dev/zenn/articles/zenn-cli-guide)  

## 執筆フォーマット

1. タイトル
   - できるだけ具体的にする（例：「Next.jsで画像最適化をする方法」）
2. 概要（Introduction）
   - この記事で扱うテーマの簡単な説明
   - 読者がこの記事を読むことで得られること
3. 背景・目的（Why / Motivation）
   - なぜこの技術を使うのか？（課題やユースケースの説明）
   - どんな問題を解決するのか？
4. 前提・環境（Prerequisites / Environment）
   - 実行環境（OS, 言語, フレームワーク, バージョンなど）
   - 必要な知識や事前準備
5. 実装・解説（Implementation / Explanation）
   - 実際の手順やコード例
   - 各ステップの説明（可能なら図やスクリーンショットを交える）
   - よくあるエラーや注意点
6. 応用・発展（Advanced / Variations）※あれば
   - さらに発展させる方法
   - 他の実装パターンや選択肢
7. まとめ（Conclusion）
   - 記事の要点を振り返る
   - この記事を読んだ人が次にできること（関連技術、参考資料など）
8. 参考資料・リンク（References）
   - 公式ドキュメント
   - 関連する技術記事
   - 自分の他の記事

## docker_oracle_dotnet

``` bash
# ×
$ cd docker-images\OracleDatabase\SingleInstance\dockerfiles\
.\buildContainerImage.sh -v 21.3.0 -x -i
# bash: cd: too many arguments

# ×
$ .\buildContainerImage.sh -v 21.3.0 -x -i
# bash: .buildContainerImage.sh: command not found

# ○
cd docker-images\OracleDatabase\SingleInstance\dockerfiles\
./buildContainerImage.sh -v 21.3.0 -x -i
```


<!-- GitHub Actionsやスクリプトを紹介している記事はあるものの、ゼロから構築する方法を詳しく解説しているサイトは見つからず…。  
当時は時間もなかったため、半ば諦めかけていました。  
しかし、どうしても実現したかったので、GitHub Actionsを一から学び、AIの力も借りながら試行錯誤した結果、なんとか実装することが出来ました。  

これらの経験を元に、ゼロベースで構築方法が分からず苦労した当時の自分のような人のために、この記事を書こうと思いました。   

私自身、GitHub Actionsの知識がほぼない状態から学びながら構築しました。  

これらの経験を元に、**ゼロベースで構築方法が分からず苦労した当時の自分のような人のため**に、この記事を書こうと思いました。
-->

<!-- 
完全にこちらの記事[(結局Githubに学習履歴を統一した方が諸々良かった)](https://zenn.dev/bun913/articles/study-history-on-github)の通りなので、そのまま引用させて頂きます。  

>私はアウトプットを大事にしたいと思いつつも以下のような悩みを持っておりました。
>
>- 毎日何かしらの学習をしているので、毎日学習履歴を残したい
>- エンジニアたるもの、どうせならGithub に草を生やしたい
>- コーディング系の学習はしていなくても、読書したり資格の学習をしている日があるが、Github のプロフィールだけ見ると何もしていないように見える
>- また資格学習のために外部サービスを使って勉強しているけど同じく何も勉強していないように見える -->

<!-- 
## なぜこの記事を書いたのか？  

NotionをヘッドレスCMSとして運用させる記事はあれど、具体的な構築方法まで解説してくれているサイトがありませんでした。(自分が無知なだけで、分かる人が見たらそれで十分なのかもしれません)  
参考サイトやAIを駆使した結果、なんとか実現する事が出来たので、自分のように何の知識もない人の参考になればと思って書きました。  
(技術記事のネタにもなるし…)  
-->

<!-- この流れが実現できたら「完璧だな」と思い、方法を調べ始めました。  
すると、**まさに求めていた仕組みを実現している人たちを発見！**  
意気揚々と実装を試みましたが、思った以上に難しく、すぐには理解できませんでした。   -->

<!-- 
元々、TIL（Today I Learned）の取り組みをしていましたが、技術スタックごとにディレクトリを分ける方法に限界を感じていました。  
特に複数の技術が関わる場合、どのディレクトリに配置すればよいのか判断に迷うことが多かったのです。  
例えば、「C#でEFCoreを使ってデータベース操作をする」という記事を書いた場合、C#？ EFCore？ データベース？ どのディレクトリが最適なのか決めかねる、といった具合です。  

何か良い方法はないかと模索していたところ、Notionのデータベース機能を活用し、タグを使って技術スタックを管理する方法を思いつきました。  
実際にNotionである程度の仕組みを構築し、運用を始めてみたのですが、一つ問題がありました。  
**学習記録はNotionに蓄積されるものの、GitHubには反映されない**ため、せっかくの学習成果が形として残らないのがもったいないと感じたのです。   -->

<!-- 
概要

この記事では、GitHub Actionsを利用して、NotionのデータをMarkdown形式で抽出し、GitHubに自動的に反映させる仕組みの構築方法を紹介します。 
-->

<!--
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


- **Notion**（記事を管理するCMS代わり）  
- **Notion API**（Notionのデータを取得する）  
- **GitHub & GitHub Actions**（記事データをMarkdownに変換し管理する）  
- **C#**（APIと連携するプログラムを作成）   


## まとめ  

- NotionをヘッドレスCMSとして活用するメリット  
- 実装の流れをおさらい  
- 今後の改善点や発展的な使い方  

### 🎯 こんな人に役立つ記事  

- **NotionをCMSとして使いたいが、構築方法が分からない人**  
- **GitHub ActionsでNotionの記事を自動管理したい人**  
- **実際に動くサンプルを試してみたい人**  

-->
