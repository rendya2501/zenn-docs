# メモ

※コマンドはzenn-docsのルートで実行すること  

記事の作成コマンド  
`npx zenn new:article`  

記事のプレビューコマンド  
`npx zenn preview`  

<!-- --- -->

## 1_docker_oracle_dotnet

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
