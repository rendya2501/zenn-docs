---
title: "【勉強メモ】DockerでOracleの環境構築 + .Net接続サンプル Ver2"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dotnet","csharp","docker","oracle"]
published: true
---

## はじめに

完全に個人用の勉強メモです。  

- Oracle学習のために環境を構築したい
- でもローカルの環境を汚したくない
- それならDockerだよね

ということで、こちらの記事([【Docker】Oracleを無料で簡単にローカルに構築する](https://zenn.dev/re24_1986/articles/29430f2f8b4b46))をほぼ参考にしつつ、躓いたところをまとめたり、立ち上げた環境に対してC#で簡単なCRUDプログラムを実行する所までをまとめました。  

因みにこの記事は、過去の記事(<https://zenn.dev/rendya/articles/docker_oracle_dotnet>)の書き直しです。  
zennの記事整理もかねて、2025年3月の環境で書き直しました。  
昔の記事は、目に余る出来栄えの為、戒めとして残しておきます。  

## 実行環境

- Win11 HOME 24H2  
- Docker version 27.5.1, build 9f9e405  
- .NET 8  
- .NET SDK インストール済み  
- VSCode  

## 構築の流れ

1. **Oracle環境構築**  
[参考サイト](https://zenn.dev/re24_1986/articles/29430f2f8b4b46)を元に`docker-compose up -d` でコンテナを起動するまで作業を行う。  
2. **サンプルプログラムの作成**  
立ち上げたコンテナに対してCRUDを行うサンプルプログラムを作成する。  

## サンプルプロジェクトの作成  

全ての操作をVSCode上で行うことで話を進めていきます。  
`docker-oracle-dotnet-sample` という名称でフォルダを作成し、VSCodeで開いてください。  
`ctrl + j` でターミナルを表示します。  
使用するターミナルは `PowerShell` とします。  

## プロジェクト構成

``` txt
docker-oracle-dotnet-sample/
├── docker-images/
│   └── etc...
├── src/
│   ├── .gitignore
│   ├── DockerOracle.csproj
│   └── Program.cs
└── docker-compose.yml
```

## 1. Oracle環境構築

### 環境構築中に躓いたこと

Windows環境でフォルダ右クリックからの `git bash` を使って、参考サイトの手順どおりに環境を構築していたのですが、**イメージ作成シェルの実行** の項目において躓きました。  
参考サイトでは以下のように記述されていますが、`git bash` ではエラーとなります。  

``` bash
cd docker-images\OracleDatabase\SingleInstance\dockerfiles\
.\buildContainerImage.sh -v 21.3.0 -x -i
```

`bash (Linux/macOS/WSL)` において、パスの指定は `/` である必要があります。  
`PowerShell (Windows)`なら`\`も`/`も使用可能なので問題ないのですが、 `git bash` では前述の通りなのでエラーとなってしまいました。  
使うターミナルとパスの関係を知っていないと地味に嵌ると思います。  
ややこしいですね…。  

一応、AIにまとめてもらった物を載せておきます。  

| 環境 | ディレクトリ区切り | カレントディレクトリの実行 |
|------|------------------|--------------------|
| **cmd.exe (Windows)** | `\` | `buildContainerImage.sh` または `.\buildContainerImage.sh` |
| **PowerShell (Windows)** | `\`（または `/`） | `.\buildContainerImage.sh`（PowerShellでは `./` も可） |
| **bash (Linux/macOS/WSL)** | `/` | `./buildContainerImage.sh`（必須） |

この問題以外は、参考サイトの通りに構築していくことが出来ました。  

## 2. サンプルプログラムの作成

### プロジェクト作成  

`srcフォルダ` に `.net8` バージョンの `Console` プロジェクトを作成します。  

``` bash
dotnet new console -f net8.0 -n DockerOracle -o src
```

### パッケージの追加

`dotnet`コマンドを使用してパッケージをプロジェクトに追加します。  
`src`フォルダに移動してコマンドを実行してください。  

``` bash
cd src
dotnet add package Dapper
dotnet add package Oracle.ManagedDataAccess.Core
```

- Dapper
  - .NET用の軽量なORMライブラリ  
- Oracle.ManagedDataAccess.Core
  - Oracleデータベースにアクセスするための.NET用のデータプロバイダ  

### gitignore

飛ばしても良い作業ですが、GitHubに上げるなら作っておいた方が良いです。  
`obj`や`bin`フォルダをgitの追跡対象から除外します。  

``` bash
dotnet new gitignore
```

### サンプルプログラム

簡単なCRUDを行うプログラムです。  
コメントを読めば大体何をしているのかは分かると思います。  

`Program.cs`にコピペしてください。  

``` cs
using Dapper;
using Oracle.ManagedDataAccess.Client;

// 接続文字列
// SYSユーザーとして接続する場合は、SYSDBAまたはSYSOPERの権限を指定する必要があります。
// 接続文字列に DBA Privilege=SYSDBA を含めることで、SYSDBA権限で接続します。
string connectionString = "User Id=sys;Password=passw0rd;DBA Privilege=SYSDBA;Data Source=(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=localhost)(PORT=1521)))(CONNECT_DATA=(SERVER=DEDICATED)(SERVICE_NAME=XE)));";

// DB接続
Console.WriteLine("Connecting to Oracle...");
using OracleConnection db = new(connectionString);
Console.WriteLine("Connection successful!");
Console.WriteLine("");

// テーブルの作成または初期化
Console.WriteLine("Create Table:");
CreateOrTruncateTable(db);
Console.WriteLine("Create Table successful!");
Console.WriteLine("");

// データの挿入
Console.WriteLine("Insert Data:");
InsertData(db, "John Doe", 30);
InsertData(db, "Hoge Fuga", 25);
InsertData(db, "Piyo Piyo", 20);
DisplayData(db);
Console.WriteLine("");

// データの更新
Console.WriteLine("Update Data:");
UpdateData(db, "UPDATEEEEEEEEEEEEEEE", 35);
DisplayData(db);
Console.WriteLine("");

// データの削除
Console.WriteLine("Delete Data:");
DeleteData(db);
DisplayData(db);
Console.WriteLine("");

// テーブルの削除
Console.WriteLine("Drop Table:");
DropTable(db);
Console.WriteLine("Drop Table successful!");

/// <summary>
/// テーブルの作成または初期化
/// </summary>
/// <param name="db"></param>
void CreateOrTruncateTable(OracleConnection db)
{
    var createTableQuery = """
        DECLARE
            table_count INTEGER;
        BEGIN
            SELECT COUNT(*) INTO table_count FROM user_tables WHERE table_name = 'SAMPLETABLE';
            IF table_count = 0 THEN
                EXECUTE IMMEDIATE 'CREATE TABLE SampleTable (
                    Id NUMBER GENERATED BY DEFAULT AS IDENTITY,
                    Name VARCHAR2(100),
                    Age NUMBER
                )';
            ELSE
                EXECUTE IMMEDIATE 'TRUNCATE TABLE SampleTable';
            END IF;
        END;
        """;
    db.Execute(createTableQuery);
}

/// <summary>
/// データの挿入
/// </summary>
/// <param name="db"></param>
/// <param name="name"></param>
/// <param name="age"></param>
void InsertData(OracleConnection db, string name, int age)
{
    var insertQuery = "INSERT INTO SampleTable (Name, Age) VALUES (:Name, :Age)";
    db.Execute(insertQuery, new { Name = name, Age = age });
}

/// <summary>
/// データの表示
/// </summary>
/// <param name="db"></param>
void DisplayData(OracleConnection db)
{
    var selectQuery = "SELECT * FROM SampleTable";
    var data = db.Query<SampleTable>(selectQuery);
    foreach (var item in data)
    {
        Console.WriteLine($"ID: {item.Id}, Name: {item.Name}, Age: {item.Age}");
    }
}

/// <summary>
/// データの更新
/// </summary>
/// <param name="db"></param>
/// <param name="name"></param>
/// <param name="age"></param>
void UpdateData(OracleConnection db, string name, int age)
{
    var selectQuery = "SELECT * FROM SampleTable";
    var data = db.Query<SampleTable>(selectQuery);
    var updateID = data.First().Id;
    var updateQuery = "UPDATE SampleTable SET Name = :Name, Age = :Age WHERE ID = :id";
    db.Execute(updateQuery, new { ID = updateID, Name = name, Age = age });
}

/// <summary>
/// データの削除
/// </summary>
/// <param name="db"></param>
void DeleteData(OracleConnection db)
{
    var selectQuery = "SELECT * FROM SampleTable";
    var data = db.Query<SampleTable>(selectQuery);
    var deleteID = data.Last().Id;
    var deleteQuery = "DELETE FROM SampleTable WHERE ID = :id";
    db.Execute(deleteQuery, new { ID = deleteID });
}

/// <summary>
/// テーブルの削除
/// </summary>
/// <param name="db"></param>
void DropTable(OracleConnection db)
{
    var createTableQuery = "DROP TABLE SampleTable";
    db.Execute(createTableQuery);
}


public class SampleTable
{
    public int Id { get; set; }
    public string? Name { get; set; }
    public int Age { get; set; }
}
```

### プログラムの実行

`dotnet` コマンドで実行します。

``` bash
dotnet run
```

以下の出力を確認できればOKです。  

``` logs
Connecting to Oracle...
Connection successful!

Create Table:
Create Table successful!

Insert Data:
ID: 1, Name: John Doe, Age: 30
ID: 2, Name: Hoge Fuga, Age: 25
ID: 3, Name: Piyo Piyo, Age: 20

Update Data:
ID: 1, Name: UPDATEEEEEEEEEEEEEEE, Age: 35
ID: 2, Name: Hoge Fuga, Age: 25
ID: 3, Name: Piyo Piyo, Age: 20

Delete Data:
ID: 1, Name: UPDATEEEEEEEEEEEEEEE, Age: 35
ID: 2, Name: Hoge Fuga, Age: 25

Drop Table:
Drop Table successful!
```

### 接続文字列の余談

以下は、ChatGPT3.5時代に生成してもらった接続文字列なのですが、これでは接続できませんでした。  

``` cs
string connectionString = "User Id=sys;Password=passw0rd;DBA Privilege=SYSDBA;Data Source=(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=localhost)(PORT=1521)))(CONNECT_DATA=(SERVER=DEDICATED)(SERVICE_NAME=XEPDB1)));";
```

この時のエラーメッセージは次の通りです。

``` logs
Unhandled exception. Oracle.ManagedDataAccess.Client.OracleException (0x80004005): ORA-28009: connection as SYS should be as SYSDBA or SYSOPER
```

見た感じ権限のエラーっぽいです。  
どうやら `User Id=sys;` を指定すると自動的に、`SYSユーザー` としてログインすることとなり、その場合、接続文字列に`DBA Privilege=SYSDBA;`を含めなければならないようです。  

`SYSユーザー` はデータベース管理者用の特権ユーザーであり、通常のアプリケーションで使用するのは推奨されないようです。  

本来であれば、事前にユーザーを作成して、そのユーザーでログインすべきですが、今回は簡単なサンプルなのでご容赦ください。  

## 参考サイト

[【Docker】Oracleを無料で簡単にローカルに構築する](https://zenn.dev/re24_1986/articles/29430f2f8b4b46)

## サンプルプログラムのリポジトリ

この記事で作成した成果物を置いてます。  

[rendya2501/docker-oracle-dotnet-sample: DockerでOracleの環境構築 + .Net接続サンプル](https://github.com/rendya2501/docker-oracle-dotnet-sample/tree/main)  
