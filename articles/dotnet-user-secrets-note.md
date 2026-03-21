---
title: "【.NET】 ユーザーシークレット 学習メモ — 仕組みからアクセスパターンまで"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dotnet","csharp","security","環境構築"]
published: true
---

## はじめに

Notion ページを Markdown に変換するツールを開発中、Notion の認証トークンを使ったテストで `dotnet user-secrets` を使う機会があったので、ついでに自分用の学習ノートとしてまとめることにしました。

アイデアや設計の方向性は自分が出し、コードの具体例やドキュメントの生成はAIとの対話を通じて形にしています。

https://github.com/rendya2501/notion-md-converter

## 1. ユーザーシークレットとは

`dotnet user-secrets` は、開発時に APIキー・接続文字列・パスワードなどの**機密情報をプロジェクトファイルの外に保存する**仕組みです。  

:::message
すぐに使い始めたい方は「[4.セットアップと基本コマンド](#4.-セットアップと基本コマンド)」へどうぞ。
:::

## 2. なぜユーザーシークレットを使うのか

### 2-1. 目的：機密情報の漏洩リスクを回避する

ユーザーシークレットの目的は**機密情報を Git などのバージョン管理に含めてしまうリスクを構造的に排除すること**です。  
「プロジェクト外に分離する」はそのための手段です。  

### 2-2. .gitignore + .env との比較

`.gitignore` で `.env` を除外する方法でも同じ目的は達成できます。  
それでもユーザーシークレットを使う優位性は以下の点にあります。  

- **構造的な安全性：** `.env` はプロジェクトディレクトリ内に存在するため、`.gitignore` の設定ミスや `git add .` の誤操作でコミットされるリスクが残ります。ユーザーシークレットはプロジェクトの**外側**に保存されるため、どのような操作をしても Git に含まれません。
- **.NET との統合：** `IConfiguration` に自動統合されるため、`appsettings.json` と同じ API でアクセスできます。`.env` を .NET で使うには追加ライブラリが必要です。
- **開発環境の明示性：** `ASPNETCORE_ENVIRONMENT=Development` のときだけ読み込まれる仕組みにより、開発専用の設定であることがコードレベルで保証されます。

一方で、`.env` が優れる場面もあります。

> 🐳 Docker Compose などでコンテナに環境変数を渡す場合や、チームで `.env.example` を共有して各自がコピーする運用はシンプルで分かりやすいです。どちらが適切かはプロジェクトの構成次第です。

## 3. 内部の仕組み

### 3-1. 保存場所

シークレットはプロジェクト外の以下のパスに `secrets.json` として保存されます。

| OS | パス |
| --- | --- |
| Windows | `%APPDATA%\Microsoft\UserSecrets\<id>\secrets.json` |
| macOS / Linux | `~/.microsoft/usersecrets/<id>/secrets.json` |

`<id>` は `.csproj` に記録される GUID（`UserSecretsId`）です。

```xml
<!-- .csproj に自動追加される -->
<PropertyGroup>
  <UserSecretsId>xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx</UserSecretsId>
</PropertyGroup>
```

この GUID によって、複数プロジェクトのシークレットが混在しないよう管理されています。  
意図的に複数プロジェクトで同じ GUID を設定すれば、シークレットを共有することもできます。  

### 3-2. secrets.json の構造

`appsettings.json` と同じ JSON 形式です。コロン区切りのキーとネスト構造は等価です。  
コマンドで登録すると内部的にフラット形式で保存されます。  
直接 `secrets.json` を編集する場合はどちらでも動作します。  

```json
// フラット形式
{
  "Api:Key": "my-secret-key",
  "ConnectionStrings:DefaultConnection": "Server=...;Password=secret"
}

// ネスト形式（どちらでも同じ結果）
{
  "Api": {
    "Key": "my-secret-key"
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=...;Password=secret"
  }
}
```

### 3-3. 設定プロバイダーとしての位置づけ

.NET の設定システム（`IConfiguration`）は複数の**設定プロバイダー**を重ね合わせる構造になっており、後から追加されたプロバイダーが優先されます。  

`WebApplication.CreateBuilder()` でのデフォルトの読み込み順（優先度 低 → 高）：

1. `appsettings.json`
2. `appsettings.{Environment}.json`
3. **ユーザーシークレット**（`Development` 環境のみ）
4. 環境変数
5. コマンドライン引数

ユーザーシークレットは `appsettings.json` を**上書きします**が、環境変数には**上書きされます**。

## 4. セットアップと基本コマンド

### コマンドを実行するディレクトリ

`.csproj` があるディレクトリで実行するのが基本です。  
別の場所から実行したい場合は `--project` オプションで指定できます。  

```bash
# プロジェクトディレクトリ内で実行
cd MyProject
dotnet user-secrets init

# 別の場所から実行する場合
dotnet user-secrets init --project ./MyProject
```

`set` や `list` などすべてのコマンドで同様に使えます。

### 初期化

```bash
dotnet user-secrets init
```

### 追加・更新

専用の update コマンドはなく、`set` が追加と更新を兼ねます。  
既存キーに `set` を実行すると上書きされます。  

```bash
dotnet user-secrets set "Api:Key" "my-secret-key"
dotnet user-secrets set "Api:BaseUrl" "https://api.example.com"
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=...;Password=secret"
```

### 一覧表示

```bash
dotnet user-secrets list
```

出力例：

```txt
Api:BaseUrl = https://api.example.com
Api:Key = my-secret-key
ConnectionStrings:DefaultConnection = Server=...;Password=secret
```

### 削除

```bash
dotnet user-secrets remove "Api:Key"
dotnet user-secrets clear  # 全削除
```

### JSON で一括登録

`secrets.json` の構造をそのまま流し込むことができます。  
JSON が不正な場合は登録時点でエラーになり、何も登録されません。  

```json
// secrets.json
{
  "Api:Key": "my-secret-key",
  "Api:BaseUrl": "https://api.example.com",
  "ConnectionStrings:DefaultConnection": "Server=...;Password=secret"
}
```

OS によってコマンドが異なります。

```bash
# macOS / Linux / Git Bash
cat secrets.json | dotnet user-secrets set

# Windows コマンドプロンプト
type secrets.json | dotnet user-secrets set
```

```powershell
# Windows PowerShell（-Raw で1つの文字列として渡す必要があります）
Get-Content secrets.json -Raw | dotnet user-secrets set

# cat は Get-Content のエイリアスのため、-Raw が必要です
cat secrets.json -Raw | dotnet user-secrets set
```

## 5. アプリの種類別：読み込みとアクセス方法

### 5-1. ASP.NET Core の場合

`ASPNETCORE_ENVIRONMENT=Development` のとき、`IConfiguration` に**自動統合**されます。追加設定は不要です。

```csharp
var builder = WebApplication.CreateBuilder(args);
// この時点で builder.Configuration にユーザーシークレットが含まれている
```

**直接取得：**

```csharp
var apiKey = builder.Configuration["Api:Key"];
var connStr = builder.Configuration.GetConnectionString("DefaultConnection");
```

**Options パターン経由（推奨）：**

以下のユーザーシークレットが登録されている場合：

```bash
dotnet user-secrets set "Api:Key" "my-secret-key"
dotnet user-secrets set "Api:BaseUrl" "https://api.example.com"
```

`GetSection("Api")` は `Api:` プレフィックスを持つキーをまとめて取得し、設定クラスの各プロパティにバインドします。  
プロパティ名がキー名に対応します。  

```csharp
// 設定クラス
public class ApiSettings
{
    public string Key { get; set; } = string.Empty;
    public string BaseUrl { get; set; } = string.Empty;
}

// Program.cs で登録
builder.Services.Configure<ApiSettings>(
    builder.Configuration.GetSection("Api"));

// DI 経由で利用
public class MyService(IOptions<ApiSettings> options)
{
    private readonly ApiSettings _settings = options.Value;

    public void DoSomething()
    {
        var key = _settings.Key;      // "my-secret-key"
        var url = _settings.BaseUrl;  // "https://api.example.com"
    }
}
```

### 5-2. コンソールアプリ・テストプロジェクトの場合

自動統合されないため、`ConfigurationBuilder` で明示的に読み込みます。  
型引数には**そのプロジェクト内の任意のクラス**を指定します（指定したクラスが属するアセンブリの `UserSecretsId` が使われます）。  

```csharp
var config = new ConfigurationBuilder()
    .AddUserSecrets<Program>()  // テストプロジェクトなら任意のクラスでよい
    .Build();
```

**直接取得：**

```csharp
var apiKey = config["Api:Key"];
var connStr = config.GetConnectionString("DefaultConnection");
```

**キーが未設定の場合に例外を出す：**

```csharp
var apiKey = config["Api:Key"]
    ?? throw new InvalidOperationException("User Secrets に Api:Key が設定されていません。");
```

`optional` 引数でキーが見つからない場合の挙動を制御することもできます。

```csharp
.AddUserSecrets<Program>(optional: true)   // UserSecretsId が未設定でも例外を出さない
.AddUserSecrets<Program>(optional: false)  // UserSecretsId が未設定なら例外を出す（デフォルト）
```

> ℹ️ `optional` のデフォルト値は .NET 6.0 で `true` から `false` に変更されました。.NET 5 以前のコードを移行する際は注意が必要です。

**Options パターン経由：**

コンソールアプリやテストプロジェクトでも Options パターンは使用できます。  
`ServiceCollection` を手動で構築することで ASP.NET Core と同じ仕組みを利用できます。  

```csharp
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Options;

var config = new ConfigurationBuilder()
    .AddUserSecrets<Program>()
    .Build();

var services = new ServiceCollection();
services.Configure<ApiSettings>(config.GetSection("Api"));

var provider = services.BuildServiceProvider();
var settings = provider.GetRequiredService<IOptions<ApiSettings>>().Value;

Console.WriteLine(settings.Key);     // "my-secret-key"
Console.WriteLine(settings.BaseUrl); // "https://api.example.com"
```

## 6. セキュリティの考え方と限界

ユーザーシークレットは開発専用の仕組みであり、暗号化は行われません。  
あくまで「誤って Git に含めない」ための仕組みである点を理解した上で使うことが重要です。  

- **暗号化されていません**。ファイルシステムにアクセスできれば読まれます。
- セキュリティ上の信頼は「プロジェクトに含まれない＝Git に乗らない」という点のみです。
- 本番機や共有サーバーでは**絶対に使わないでください**。

## 7. まとめ

ユーザーシークレットは「機密情報を Git に含めない」という**開発フローの安全弁**として機能します。`.gitignore + .env` と比べた最大の優位性は、プロジェクト外に保存されるという**構造的な安全性**です。あくまでローカル開発専用の仕組みであり、本番環境では別途シークレット管理サービスを使うことが前提です。

## 参考

- [Safe storage of app secrets in development in ASP.NET Core | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/security/app-secrets)
- [Storing application secrets safely during development - .NET Microservices | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/secure-net-microservices-web-applications/developer-app-secrets-storage)
- [Microsoft.Extensions.Configuration.UserSecrets | NuGet](https://www.nuget.org/packages/Microsoft.Extensions.Configuration.UserSecrets)
