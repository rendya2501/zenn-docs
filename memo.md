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
