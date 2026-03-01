# docker-oracle-dotnet

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
