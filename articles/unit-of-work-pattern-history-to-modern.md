---
title: "Unit of Work パターン研究ノート：歴史から現代の実装まで"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp","dotnet","designpattern","architecture"]
published: false
---

## はじめに

この記事は、自分のための研究ノートです。

記事を書くきっかけは、SQLを書く場所もトランザクション境界も曖昧なオレオレフレームワークを実装したプロジェクトに参加して、炎上したことです。「設計がダメなのは分かったけど、どうすればもっとマシになるのか？」という問いから、リポジトリパターンやUnit of Workパターンにたどり着きましたが、当時は勉強する時間が取れず、ずっと積み残したままでした。

時間ができて、AIと一緒に調べ直してみると、断片的に知っていたことが一本の線でつながっていきました。サンプルコードを作りながらAIに評価してもらう中で、「UoWの定義から現代までの変遷をまとめたらどうなるか」と試してみたら、想像以上に面白い記事になりました。アイデアや設計の方向性は自分が出し、コードの具体化はAIとの対話を通じて形にしています。

まとめたらそれなりのものになっただけで、よかったら誰か読んでみてください程度のものです。特にUoWについて調べている人や、レガシー案件でトランザクション管理に悩んでいる人の参考になれば幸いです。

## TL;DR

- Unit of Workパターンの原典定義（Fowler）と、現代での責務の変化
- EF Core環境でUoWが不要になった理由
- Dapper/生SQL環境向けの3つの実装パターン（ベーシックUoW・スコープベースUoW・パイプラインUoW）  
  ※スコープベースUoWはResult型 + ラムダスコープによる筆者の実装

## Unit of Work とは

Martin Fowlerが『Patterns of Enterprise Application Architecture』(2002) で提唱したデザインパターンです。

> ビジネストランザクションに影響を受けるオブジェクトのリストを保持し、変更の書き込みと並行性問題の解決を調整する

原典では3つの責務を持っていました。

1. **変更追跡（Change Tracking）**：どのオブジェクトが変更されたか追跡する
2. **Identity Map**：同一オブジェクトの重複読み込みを防ぐ
3. **トランザクション管理**：複数の変更を1つの単位としてコミットする

ただし、この定義は2002年のものです。現代においてUoWが解決しようとしている問題は、より絞られています。

**現代のUoWが解決する問題：**  

- 複数のリポジトリ操作を1つのトランザクションとして扱いたい
- データの一貫性を保証したい
- トランザクション境界を明示的に管理したい
- ボイラープレートなコードを排除したい

なぜ定義が変わったのか――その理由を理解するために、まず歴史から見ていきます。

## Part 1：古典的な Unit of Work（ADO.NET時代）

### 時代背景（2000年代初頭〜2008年頃）

**技術環境**  

- .NET Framework 1.0〜2.0 の時代
- ADO.NET が主流のデータアクセス技術
- ORM は NHibernate がほぼ唯一の選択肢（未成熟）
- DIコンテナは存在しない、または実験的段階（Spring.NET、Castle Windsor）
- デザインパターンブーム：GoFパターン、Fowlerのエンタープライズパターンが注目される

**言語的制約（C# 1.0〜2.0）**  

- ジェネリクスがない、または登場したばかり
- ラムダ式なし（C# 3.0で登場）
- `var` キーワードなし（C# 3.0で登場）
- オブジェクト初期化子なし（C# 3.0で登場）
- 非同期は `BeginXxx/EndXxx` のAPMパターン（`async/await` は C# 5.0で登場）
- null 許容参照型なし

### なぜ「古典的UoW」はこうなったのか

DIコンテナがないため、**UoWが自らリポジトリを生成・管理する必要がありました。** これは単純なバッドプラクティスではなく、技術的制約への合理的な適応です。

```txt
[ Client ]
    ↓ 
[ Unit of Work ] ─── (生成・保持) ──→ [ Repository A ]
    │            ─── (生成・保持) ──→ [ Repository B ]
    ↓
[ DB Connection / Transaction ]
```

UoWが「Factory」と「Transaction Manager」の2つの責務を担う設計になるのは、この状況では必然でした。

### Fowler が定義した UoW の3責務を実装する

原典のUoWは変更追跡・Identity Map・トランザクション管理の3つを担います。以下はその全実装です。C# 2.0時代の書き方（`var`なし、オブジェクト初期化子なし、ジェネリクスは限定的）で記述します。

#### Identity Map とは

同一トランザクション内で、同じIDのオブジェクトを複数回DBから取得しないための仕組みです。一度取得したオブジェクトを `Dictionary` で保持し、再要求時はDBを叩かずキャッシュから返します。

```csharp
// Identity Mapなし（同じIDを2回取得するとDBを2回叩く）
OrderRepository repo = new OrderRepository(connection, transaction);
Order order1 = repo.GetById(1); // DB問い合わせ
Order order2 = repo.GetById(1); // また DB問い合わせ
// order1 と order2 は別インスタンス

// Identity Mapあり（2回目以降はキャッシュから返す）
Order order1 = uow.GetOrder(1); // DB問い合わせ
Order order2 = uow.GetOrder(1); // マップから返す（DB問い合わせなし）
// order1 と order2 は同一インスタンス
```

#### 変更追跡（Dirty Tracking）とは

`RegisterClean()` で「取得したオブジェクトの元の状態」を記録し、`Commit()` 時に現在の状態と比較して変更があったものだけをUPDATEします。手動で `Update()` を呼ぶ必要がなくなります。

#### 完全実装

```csharp
// エンティティの基底クラス（C# 2.0スタイル）
public abstract class Entity
{
    private int _id;

    public int Id
    {
        get { return _id; }
        set { _id = value; }
    }
}

public class Order : Entity
{
    private int _customerId;
    private DateTime _orderDate;

    public int CustomerId
    {
        get { return _customerId; }
        set { _customerId = value; }
    }

    public DateTime OrderDate
    {
        get { return _orderDate; }
        set { _orderDate = value; }
    }
}

public class Customer : Entity
{
    private string _name;
    private DateTime _lastOrderDate;

    public string Name
    {
        get { return _name; }
        set { _name = value; }
    }

    public DateTime LastOrderDate
    {
        get { return _lastOrderDate; }
        set { _lastOrderDate = value; }
    }
}

public class Product : Entity
{
    private string _name;
    private int _stock;

    public string Name
    {
        get { return _name; }
        set { _name = value; }
    }

    public int Stock
    {
        get { return _stock; }
        set { _stock = value; }
    }
}

// -------------------------------------------------------
// Identity Map + 変更追跡 + トランザクション管理 を持つ UoW
// -------------------------------------------------------
public interface IUnitOfWork : IDisposable
{
    // リポジトリへのアクセス（Factory兼任）
    IOrderRepository Orders { get; }
    ICustomerRepository Customers { get; }
    IProductRepository Products { get; }

    // トランザクション管理
    void BeginTransaction();
    void Commit();
    void Rollback();

    // Identity Map 操作
    void RegisterClean(Entity entity);
    void RegisterNew(Entity entity);
    void RegisterDirty(Entity entity);
    void RegisterDeleted(Entity entity);
}

public class UnitOfWork : IUnitOfWork
{
    private IDbConnection _connection;
    private IDbTransaction _transaction;

    // リポジトリ（遅延生成）
    private IOrderRepository _orders;
    private ICustomerRepository _customers;
    private IProductRepository _products;

    // Identity Map：型名 + ID をキーにオブジェクトをキャッシュ
    private Dictionary<string, Entity> _identityMap;

    // 変更追跡リスト
    private List<Entity> _newEntities;
    private List<Entity> _dirtyEntities;
    private List<Entity> _deletedEntities;

    public UnitOfWork(string connectionString)
    {
        _connection = new SqlConnection(connectionString);
        _identityMap = new Dictionary<string, Entity>();
        _newEntities = new List<Entity>();
        _dirtyEntities = new List<Entity>();
        _deletedEntities = new List<Entity>();
    }

    // -------------------------------------------------------
    // リポジトリへのアクセス（Factory 責務）
    // -------------------------------------------------------

    public IOrderRepository Orders
    {
        get
        {
            if (_orders == null)
            {
                _orders = new OrderRepository(_connection, _transaction, this);
            }
            return _orders;
        }
    }

    public ICustomerRepository Customers
    {
        get
        {
            if (_customers == null)
            {
                _customers = new CustomerRepository(_connection, _transaction, this);
            }
            return _customers;
        }
    }

    public IProductRepository Products
    {
        get
        {
            if (_products == null)
            {
                _products = new ProductRepository(_connection, _transaction, this);
            }
            return _products;
        }
    }

    // -------------------------------------------------------
    // Identity Map 操作
    // -------------------------------------------------------

    private string GetKey(Entity entity)
    {
        return entity.GetType().Name + "_" + entity.Id;
    }

    private string GetKey(Type type, int id)
    {
        return type.Name + "_" + id;
    }

    // 「取得済みのクリーンなオブジェクト」として登録
    public void RegisterClean(Entity entity)
    {
        string key = GetKey(entity);
        if (!_identityMap.ContainsKey(key))
        {
            _identityMap[key] = entity;
        }
    }

    // 新規追加オブジェクトとして登録
    public void RegisterNew(Entity entity)
    {
        _newEntities.Add(entity);
    }

    // 変更ありオブジェクトとして登録
    public void RegisterDirty(Entity entity)
    {
        if (!_dirtyEntities.Contains(entity) && !_newEntities.Contains(entity))
        {
            _dirtyEntities.Add(entity);
        }
    }

    // 削除対象として登録
    public void RegisterDeleted(Entity entity)
    {
        if (_newEntities.Contains(entity))
        {
            _newEntities.Remove(entity);
            return;
        }
        if (!_deletedEntities.Contains(entity))
        {
            _deletedEntities.Add(entity);
        }
    }

    // Identity Map からオブジェクトを検索（リポジトリが呼び出す）
    public Entity FindInMap(Type type, int id)
    {
        string key = GetKey(type, id);
        if (_identityMap.ContainsKey(key))
        {
            return _identityMap[key];
        }
        return null;
    }

    // -------------------------------------------------------
    // トランザクション管理
    // -------------------------------------------------------

    public void BeginTransaction()
    {
        if (_transaction != null)
        {
            throw new InvalidOperationException("Transaction already started");
        }

        if (_connection.State != ConnectionState.Open)
        {
            _connection.Open();
        }

        _transaction = _connection.BeginTransaction();

        // トランザクション開始時にリポジトリへ最新のトランザクションを伝播
        // （遅延生成済みの場合のみ）
        if (_orders != null)
        {
            ((OrderRepository)_orders).UpdateTransaction(_transaction);
        }
        if (_customers != null)
        {
            ((CustomerRepository)_customers).UpdateTransaction(_transaction);
        }
        if (_products != null)
        {
            ((ProductRepository)_products).UpdateTransaction(_transaction);
        }
    }

    public void Commit()
    {
        if (_transaction == null)
        {
            throw new InvalidOperationException("No active transaction");
        }

        try
        {
            // 変更追跡：新規・更新・削除を一括処理
            CommitNewEntities();
            CommitDirtyEntities();
            CommitDeletedEntities();

            _transaction.Commit();
        }
        catch
        {
            _transaction.Rollback();
            throw;
        }
        finally
        {
            _transaction.Dispose();
            _transaction = null;
            ClearTracking();
        }
    }

    public void Rollback()
    {
        if (_transaction == null)
        {
            return;
        }

        try
        {
            _transaction.Rollback();
        }
        finally
        {
            _transaction.Dispose();
            _transaction = null;
            ClearTracking();
        }
    }

    // -------------------------------------------------------
    // 変更追跡：Commit時に呼ばれる内部処理
    // -------------------------------------------------------

    private void CommitNewEntities()
    {
        for (int i = 0; i < _newEntities.Count; i++)
        {
            Entity entity = _newEntities[i];

            if (entity is Order)
            {
                InsertOrder((Order)entity);
            }
            else if (entity is Customer)
            {
                InsertCustomer((Customer)entity);
            }
            else if (entity is Product)
            {
                InsertProduct((Product)entity);
            }
        }
    }

    private void CommitDirtyEntities()
    {
        for (int i = 0; i < _dirtyEntities.Count; i++)
        {
            Entity entity = _dirtyEntities[i];

            if (entity is Order)
            {
                UpdateOrder((Order)entity);
            }
            else if (entity is Customer)
            {
                UpdateCustomer((Customer)entity);
            }
            else if (entity is Product)
            {
                UpdateProduct((Product)entity);
            }
        }
    }

    private void CommitDeletedEntities()
    {
        for (int i = 0; i < _deletedEntities.Count; i++)
        {
            Entity entity = _deletedEntities[i];
            DeleteEntity(entity);
        }
    }

    private void ClearTracking()
    {
        _newEntities.Clear();
        _dirtyEntities.Clear();
        _deletedEntities.Clear();
    }

    // -------------------------------------------------------
    // SQL実行ヘルパー（実際はリポジトリに持たせる場合もある）
    // -------------------------------------------------------

    private void InsertOrder(Order order)
    {
        IDbCommand cmd = _connection.CreateCommand();
        cmd.Transaction = _transaction;
        cmd.CommandText =
            "INSERT INTO Orders (CustomerId, OrderDate) " +
            "VALUES (@CustomerId, @OrderDate)";

        IDbDataParameter p1 = cmd.CreateParameter();
        p1.ParameterName = "@CustomerId";
        p1.Value = order.CustomerId;
        cmd.Parameters.Add(p1);

        IDbDataParameter p2 = cmd.CreateParameter();
        p2.ParameterName = "@OrderDate";
        p2.Value = order.OrderDate;
        cmd.Parameters.Add(p2);

        cmd.ExecuteNonQuery();
    }

    private void UpdateOrder(Order order)
    {
        IDbCommand cmd = _connection.CreateCommand();
        cmd.Transaction = _transaction;
        cmd.CommandText =
            "UPDATE Orders SET CustomerId = @CustomerId, " +
            "OrderDate = @OrderDate WHERE Id = @Id";

        IDbDataParameter p1 = cmd.CreateParameter();
        p1.ParameterName = "@CustomerId";
        p1.Value = order.CustomerId;
        cmd.Parameters.Add(p1);

        IDbDataParameter p2 = cmd.CreateParameter();
        p2.ParameterName = "@OrderDate";
        p2.Value = order.OrderDate;
        cmd.Parameters.Add(p2);

        IDbDataParameter p3 = cmd.CreateParameter();
        p3.ParameterName = "@Id";
        p3.Value = order.Id;
        cmd.Parameters.Add(p3);

        cmd.ExecuteNonQuery();
    }

    private void InsertCustomer(Customer customer)
    {
        IDbCommand cmd = _connection.CreateCommand();
        cmd.Transaction = _transaction;
        cmd.CommandText =
            "INSERT INTO Customers (Name, LastOrderDate) " +
            "VALUES (@Name, @LastOrderDate)";

        IDbDataParameter p1 = cmd.CreateParameter();
        p1.ParameterName = "@Name";
        p1.Value = customer.Name;
        cmd.Parameters.Add(p1);

        IDbDataParameter p2 = cmd.CreateParameter();
        p2.ParameterName = "@LastOrderDate";
        p2.Value = customer.LastOrderDate;
        cmd.Parameters.Add(p2);

        cmd.ExecuteNonQuery();
    }

    private void UpdateCustomer(Customer customer)
    {
        IDbCommand cmd = _connection.CreateCommand();
        cmd.Transaction = _transaction;
        cmd.CommandText =
            "UPDATE Customers SET Name = @Name, " +
            "LastOrderDate = @LastOrderDate WHERE Id = @Id";

        IDbDataParameter p1 = cmd.CreateParameter();
        p1.ParameterName = "@Name";
        p1.Value = customer.Name;
        cmd.Parameters.Add(p1);

        IDbDataParameter p2 = cmd.CreateParameter();
        p2.ParameterName = "@LastOrderDate";
        p2.Value = customer.LastOrderDate;
        cmd.Parameters.Add(p2);

        IDbDataParameter p3 = cmd.CreateParameter();
        p3.ParameterName = "@Id";
        p3.Value = customer.Id;
        cmd.Parameters.Add(p3);

        cmd.ExecuteNonQuery();
    }

    private void InsertProduct(Product product)
    {
        IDbCommand cmd = _connection.CreateCommand();
        cmd.Transaction = _transaction;
        cmd.CommandText =
            "INSERT INTO Products (Name, Stock) VALUES (@Name, @Stock)";

        IDbDataParameter p1 = cmd.CreateParameter();
        p1.ParameterName = "@Name";
        p1.Value = product.Name;
        cmd.Parameters.Add(p1);

        IDbDataParameter p2 = cmd.CreateParameter();
        p2.ParameterName = "@Stock";
        p2.Value = product.Stock;
        cmd.Parameters.Add(p2);

        cmd.ExecuteNonQuery();
    }

    private void UpdateProduct(Product product)
    {
        IDbCommand cmd = _connection.CreateCommand();
        cmd.Transaction = _transaction;
        cmd.CommandText =
            "UPDATE Products SET Name = @Name, Stock = @Stock WHERE Id = @Id";

        IDbDataParameter p1 = cmd.CreateParameter();
        p1.ParameterName = "@Name";
        p1.Value = product.Name;
        cmd.Parameters.Add(p1);

        IDbDataParameter p2 = cmd.CreateParameter();
        p2.ParameterName = "@Stock";
        p2.Value = product.Stock;
        cmd.Parameters.Add(p2);

        IDbDataParameter p3 = cmd.CreateParameter();
        p3.ParameterName = "@Id";
        p3.Value = product.Id;
        cmd.Parameters.Add(p3);

        cmd.ExecuteNonQuery();
    }

    private void DeleteEntity(Entity entity)
    {
        // ⚠️ 教育目的のサンプルコードです。テーブル名を動的に生成するこの実装は
        // SQLインジェクションの危険があります。実際のコードでは固定のテーブル名を使ってください。
        string tableName = entity.GetType().Name + "s"; // 簡易的な複数形
        IDbCommand cmd = _connection.CreateCommand();
        cmd.Transaction = _transaction;
        cmd.CommandText = "DELETE FROM " + tableName + " WHERE Id = @Id";

        IDbDataParameter p = cmd.CreateParameter();
        p.ParameterName = "@Id";
        p.Value = entity.Id;
        cmd.Parameters.Add(p);

        cmd.ExecuteNonQuery();
    }

    public void Dispose()
    {
        if (_transaction != null)
        {
            _transaction.Rollback();
            _transaction.Dispose();
        }

        if (_connection != null)
        {
            _connection.Dispose();
        }
    }
}
```

#### リポジトリ側の実装（Identity Map を使う）

リポジトリの `GetById` は、まずUoWのIdentity Mapを確認し、なければDBへ問い合わせます。取得後は `RegisterClean()` でマップに登録します。

```csharp
public interface IOrderRepository
{
    Order GetById(int id);
    void Add(Order order);
    void Update(Order order);
    void Delete(Order order);
}

public class OrderRepository : IOrderRepository
{
    private IDbConnection _connection;
    private IDbTransaction _transaction;
    private UnitOfWork _uow; // Identity Map へのアクセス用

    public OrderRepository(
        IDbConnection connection,
        IDbTransaction transaction,
        UnitOfWork uow)
    {
        _connection = connection;
        _transaction = transaction;
        _uow = uow;
    }

    // UoWからトランザクションが更新されたとき呼ばれる
    public void UpdateTransaction(IDbTransaction transaction)
    {
        _transaction = transaction;
    }

    public Order GetById(int id)
    {
        // まず Identity Map を確認
        Entity cached = _uow.FindInMap(typeof(Order), id);
        if (cached != null)
        {
            return (Order)cached; // DBを叩かずに返す
        }

        // キャッシュになければDBへ
        IDbCommand cmd = _connection.CreateCommand();
        cmd.Transaction = _transaction;
        cmd.CommandText =
            "SELECT Id, CustomerId, OrderDate FROM Orders WHERE Id = @Id";

        IDbDataParameter p = cmd.CreateParameter();
        p.ParameterName = "@Id";
        p.Value = id;
        cmd.Parameters.Add(p);

        IDataReader reader = cmd.ExecuteReader();
        Order order = null;

        if (reader.Read())
        {
            order = new Order();
            order.Id = (int)reader["Id"];
            order.CustomerId = (int)reader["CustomerId"];
            order.OrderDate = (DateTime)reader["OrderDate"];
        }
        reader.Close();

        if (order != null)
        {
            _uow.RegisterClean(order); // Identity Map に登録
        }

        return order;
    }

    public void Add(Order order)
    {
        _uow.RegisterNew(order); // 新規としてマークするだけ（SQL実行はCommit時）
    }

    public void Update(Order order)
    {
        _uow.RegisterDirty(order); // 変更ありとしてマーク（SQL実行はCommit時）
    }

    public void Delete(Order order)
    {
        _uow.RegisterDeleted(order); // 削除対象としてマーク（SQL実行はCommit時）
    }
}

// CustomerRepository・ProductRepository も同様の構造
public interface ICustomerRepository
{
    Customer GetById(int id);
    void Add(Customer customer);
    void Update(Customer customer);
    void Delete(Customer customer);
}

public class CustomerRepository : ICustomerRepository
{
    private IDbConnection _connection;
    private IDbTransaction _transaction;
    private UnitOfWork _uow;

    public CustomerRepository(
        IDbConnection connection,
        IDbTransaction transaction,
        UnitOfWork uow)
    {
        _connection = connection;
        _transaction = transaction;
        _uow = uow;
    }

    public void UpdateTransaction(IDbTransaction transaction)
    {
        _transaction = transaction;
    }

    public Customer GetById(int id)
    {
        Entity cached = _uow.FindInMap(typeof(Customer), id);
        if (cached != null)
        {
            return (Customer)cached;
        }

        IDbCommand cmd = _connection.CreateCommand();
        cmd.Transaction = _transaction;
        cmd.CommandText =
            "SELECT Id, Name, LastOrderDate FROM Customers WHERE Id = @Id";

        IDbDataParameter p = cmd.CreateParameter();
        p.ParameterName = "@Id";
        p.Value = id;
        cmd.Parameters.Add(p);

        IDataReader reader = cmd.ExecuteReader();
        Customer customer = null;

        if (reader.Read())
        {
            customer = new Customer();
            customer.Id = (int)reader["Id"];
            customer.Name = (string)reader["Name"];
            customer.LastOrderDate = (DateTime)reader["LastOrderDate"];
        }
        reader.Close();

        if (customer != null)
        {
            _uow.RegisterClean(customer);
        }

        return customer;
    }

    public void Add(Customer customer)    { _uow.RegisterNew(customer); }
    public void Update(Customer customer) { _uow.RegisterDirty(customer); }
    public void Delete(Customer customer) { _uow.RegisterDeleted(customer); }
}

public interface IProductRepository
{
    Product GetById(int id);
    void Add(Product product);
    void Update(Product product);
    void Delete(Product product);
}

public class ProductRepository : IProductRepository
{
    private IDbConnection _connection;
    private IDbTransaction _transaction;
    private UnitOfWork _uow;

    public ProductRepository(
        IDbConnection connection,
        IDbTransaction transaction,
        UnitOfWork uow)
    {
        _connection = connection;
        _transaction = transaction;
        _uow = uow;
    }

    public void UpdateTransaction(IDbTransaction transaction)
    {
        _transaction = transaction;
    }

    public Product GetById(int id)
    {
        Entity cached = _uow.FindInMap(typeof(Product), id);
        if (cached != null)
        {
            return (Product)cached;
        }

        IDbCommand cmd = _connection.CreateCommand();
        cmd.Transaction = _transaction;
        cmd.CommandText =
            "SELECT Id, Name, Stock FROM Products WHERE Id = @Id";

        IDbDataParameter p = cmd.CreateParameter();
        p.ParameterName = "@Id";
        p.Value = id;
        cmd.Parameters.Add(p);

        IDataReader reader = cmd.ExecuteReader();
        Product product = null;

        if (reader.Read())
        {
            product = new Product();
            product.Id = (int)reader["Id"];
            product.Name = (string)reader["Name"];
            product.Stock = (int)reader["Stock"];
        }
        reader.Close();

        if (product != null)
        {
            _uow.RegisterClean(product);
        }

        return product;
    }

    public void Add(Product product)    { _uow.RegisterNew(product); }
    public void Update(Product product) { _uow.RegisterDirty(product); }
    public void Delete(Product product) { _uow.RegisterDeleted(product); }
}
```

### 使用例

```csharp
string connectionString = ConfigurationManager
    .ConnectionStrings["Default"].ConnectionString;

UnitOfWork uow = new UnitOfWork(connectionString);
try
{
    uow.BeginTransaction();

    // 新規注文（Commit時にINSERT）
    Order order = new Order();
    order.CustomerId = 1;
    order.OrderDate = DateTime.Now;
    uow.Orders.Add(order); // RegisterNew → Commit時にInsertOrder

    // 顧客を取得して更新（2回取得しても DBは1回しか叩かない）
    Customer customer = uow.Customers.GetById(1); // DB問い合わせ
    Customer same    = uow.Customers.GetById(1);   // Identity Mapから返す
    customer.LastOrderDate = DateTime.Now;
    uow.Customers.Update(customer); // RegisterDirty → Commit時にUpdateCustomer

    // 在庫を減らす
    int productId = 1; // 実際には注文データから取得する
    Product product = uow.Products.GetById(productId);
    product.Stock = product.Stock - 1;
    uow.Products.Update(product);

    uow.Commit(); // ここで一括SQL実行 → トランザクションコミット
}
catch (Exception ex)
{
    uow.Rollback();
    throw;
}
finally
{
    uow.Dispose();
}
```

### このパターンの評価

**メリット（当時の文脈では）：**  

- Fowlerの原典に忠実な3責務（変更追跡・Identity Map・トランザクション管理）を実装
- リポジトリ間でトランザクションが自動共有される
- 同一トランザクション内でのDBアクセスを最小化できる

**デメリット（現代から見ると）：**  

- UoWがすべてのリポジトリに密結合
- リポジトリ追加のたびにUoWインターフェースの修正が必要
- UoWが「Factory」「Transaction Manager」「変更追跡エンジン」を兼任しすぎ
- テスト・モックが非常に複雑

これらは設計ミスではなく、DIコンテナがない時代への合理的な適応です。当時のコードを読んで「なぜこんな密結合な設計を？」と思わず、「DIコンテナがなかった時代の設計だ」と理解することが重要です。

## Part 2：Entity Framework の登場と UoW の変容

### 時代背景（2008年〜2015年頃）

**技術環境**  

- .NET Framework 3.5〜4.5
- **Entity Framework 1.0 登場（2008年）**：Microsoftの公式ORM
- DIコンテナの普及：Autofac、Ninject、Unity が広く使われる
- LINQ to SQL の登場と終焉

**言語の進化（C# 3.0〜5.0）**  

- C# 3.0（2007年）：`var`、LINQ、ラムダ式、拡張メソッド、オブジェクト初期化子、自動実装プロパティ
- C# 4.0（2010年）：dynamic型、名前付き引数
- C# 5.0（2012年）：**`async/await`**（非同期プログラミングの革命）

### これが最大の転換点

2008年にEntity Framework 1.0が登場し、2012年頃に成熟したことで、.NETのデータアクセス設計は根本的に変わりました。

**`DbContext` が、UoWの3つの責務をすべて内包したのです。**

```txt
Fowlerの定義（2002年）：
┌─────────────────────────┐
│ Unit of Work            │
│ ・変更追跡              │
│ ・Identity Map          │
│ ・トランザクション管理  │
└─────────────────────────┘

Entity Framework登場後：
┌──────────────────────────┐
│ DbContext（EF / EF Core）│
│ ・変更追跡（自動）       │
│ ・Identity Map（自動）   │
│ ・SaveChanges でコミット │
└──────────────────────────┘
```

Entity Frameworkの内部動作：

1. `DbSet<T>.Add()` でエンティティを追跡開始（状態：Added）
2. エンティティのプロパティ変更を検知（状態：Modified）
3. `SaveChanges()` 呼び出し時に自動でトランザクション開始→SQL生成→コミット

つまり、**Entity Framework 自体が UoW の実装であるため、別途 UoW を実装する必要性が薄れました。**

### DIコンテナとの統合で完結する

ASP.NET MVCとDIコンテナで DbContext をスコープ管理すれば、1リクエスト1コンテキストが保証されます。C# 5.0（`async/await`）時代の書き方で記述します。

```csharp
// DbContext の定義
public class ApplicationDbContext : DbContext
{
    public DbSet<Order> Orders { get; set; }
    public DbSet<Customer> Customers { get; set; }
    public DbSet<Product> Products { get; set; }

    public ApplicationDbContext(string connectionString)
        : base(connectionString)
    {
    }
}

// DI登録（Unity の例）
public static void RegisterTypes(IUnityContainer container)
{
    container.RegisterType<ApplicationDbContext>(
        new HierarchicalLifetimeManager());

    container.RegisterType<IOrderService, OrderService>();
}

// サービス層での使用（C# 5.0 async/await）
public class OrderService : IOrderService
{
    private readonly ApplicationDbContext _context;

    public OrderService(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task CreateOrderAsync(Order order)
    {
        _context.Orders.Add(order);

        var customer = await _context.Customers.FindAsync(order.CustomerId);
        customer.LastOrderDate = DateTime.Now;

        await _context.SaveChangesAsync(); // これだけでOK
    }
}
```

### Identity Map が自動化された

Part 1で手動実装していたIdentity Mapも、Entity Frameworkでは自動提供されます。

```csharp
// 同一DbContextスコープ内
var order1 = await context.Orders.FindAsync(1); // DB問い合わせ
var order2 = await context.Orders.FindAsync(1); // マップから返す（DB問い合わせなし）

bool isSame = object.ReferenceEquals(order1, order2); // True
```

Part 1で数百行かけて手動実装していたものが、Entity Frameworkでは自動で提供されます。これがUoWの定義が変化した本質です。変更追跡とIdentity Mapが自動化されたことで、アプリケーション層がUoWに求める責務は「トランザクション境界の管理」だけになりました。

> **コラム：Entity Framework Core（EF Core）について**
>
> Part 2で扱うのは2008年登場の Entity Framework（EF）ですが、後継の **Entity Framework Core（EF Core）** についても触れておきます。
>
> EF Core は2016年に .NET Core と同時にリリースされた、Entity Frameworkの完全な書き直し版です。クロスプラットフォーム対応・オープンソース化・パフォーマンス改善が主な特徴で、現在の .NET 開発における標準的なORMです。
>
> DbContextの基本的な動作（変更追跡・Identity Map・SaveChanges）はEFからEF Coreに引き継がれており、「DbContextがUoWを内包する」という本質は変わりません。Part 3以降では EF Core を前提に解説します。

### Repository パターンとの関係

この世代では「Repository パターンは必要か？」という議論が盛んになりました。

**反対派の意見：**  

- DbContext 自体が Repository + UoW の役割を果たす
- 抽象化のレイヤーが増えすぎる
- テストはモックではなく In-Memory Database で行える

**賛成派の意見：**  

- ドメインロジックをORMから分離したい
- クエリロジックを集約したい
- 将来的なORM変更に備えたい

結果として **Generic Repository パターン**が流行しました。C# 3.0以降（ジェネリクス・LINQ・自動実装プロパティが使える時代）の書き方で記述します。

```csharp
public interface IRepository<T> where T : class
{
    T GetById(int id);
    IEnumerable<T> GetAll();
    void Add(T entity);
    void Update(T entity);
    void Delete(T entity);
}

public class Repository<T> : IRepository<T> where T : class
{
    private readonly DbContext _context;
    private readonly DbSet<T> _dbSet;

    public Repository(DbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public T GetById(int id)
    {
        return _dbSet.Find(id);
    }

    public IEnumerable<T> GetAll()
    {
        return _dbSet.ToList();
    }

    public void Add(T entity)
    {
        _dbSet.Add(entity);
    }

    public void Update(T entity)
    {
        _context.Entry(entity).State = EntityState.Modified;
    }

    public void Delete(T entity)
    {
        _dbSet.Remove(entity);
    }
}

// UoWとの統合
public interface IUnitOfWork : IDisposable
{
    IRepository<Order> Orders { get; }
    IRepository<Customer> Customers { get; }
    IRepository<Product> Products { get; }
    int SaveChanges();
}

public class UnitOfWork : IUnitOfWork
{
    private readonly ApplicationDbContext _context;
    private IRepository<Order> _orders;
    private IRepository<Customer> _customers;
    private IRepository<Product> _products;

    public UnitOfWork(ApplicationDbContext context)
    {
        _context = context;
    }

    public IRepository<Order> Orders
    {
        get
        {
            if (_orders == null)
            {
                _orders = new Repository<Order>(_context);
            }
            return _orders;
        }
    }

    public IRepository<Customer> Customers
    {
        get
        {
            if (_customers == null)
            {
                _customers = new Repository<Customer>(_context);
            }
            return _customers;
        }
    }

    public IRepository<Product> Products
    {
        get
        {
            if (_products == null)
            {
                _products = new Repository<Product>(_context);
            }
            return _products;
        }
    }

    public int SaveChanges()
    {
        return _context.SaveChanges();
    }

    public void Dispose()
    {
        _context.Dispose();
    }
}
```

### Generic Repository + UoW という罠

一見きれいに見えるこの設計ですが、実態は**DbContextをラップしているだけで、何も新しい価値を生み出していません。**

`DbContext.Set<T>()` と `SaveChanges()` を薄く包んだだけのUoWクラスは、テストのしやすさも向上せず、むしろ間接層が増えてコードが複雑になります。Entity Framework 自体が `UseInMemoryDatabase()` によるテストをサポートしているため、モックのためにリポジトリ抽象化が必要という理由も弱くなりました。

**この時代の教訓：**  

- DbContextが変更追跡・Identity Map・トランザクション管理を内包している
- Generic Repositoryは過度な抽象化になりやすい
- 「Repository パターンは Entity Framework には不要」という意見が主流になった

## Part 3：UoWを使うべき場面（EF登場以降）

Entity Framework（およびEF Core）が登場したことで、「UoWはもう不要なのか？」という議論が生まれました。結論としては、EF / EF Core を使っているなら、ほとんどの場合、不要です。DbContext 自体がUoWの役割を担っており、変更追跡・Identity Map・トランザクション管理のすべてを内包しているため、別途UoWクラスを作る必要はありません。

しかし現実には、EFを導入できない・したくない現場は多数存在します。

**EF / EF Core を導入できない現実的な理由：**  

- 既存の生SQL・ストアドプロシージャ資産が多く、EFへの置き換えコストが高い
- パフォーマンス要件が厳しく、EFの変更追跡オーバーヘッドを避けたい
- DBAが管理するストアドプロシージャとアプリケーションコードを分離したい
- EFが登場した2008年〜2016年頃は、まだEFの品質・パフォーマンスへの信頼が低かった

そうした環境では、依然として**生クエリ・Dapperによるトランザクション管理の仕組みが必要**でした。また、複数のDbContextをまたがるトランザクションや、DapperとEF Coreが混在する構成でも、明示的なトランザクション管理が求められます。

EF登場以降のUoWが担う責務は「トランザクション境界の抽象化」に絞られています。

```txt
EF登場以降のUoW：
┌─────────────────────────┐
│ Unit of Work            │
│ ・トランザクション管理  │ ← これだけに特化
└─────────────────────────┘

変更追跡とIdentity Mapは：
・EF / EF Core → 自動で提供
・Dapper       → そもそも不要（シンプルなクエリ実行に徹する）
```

なぜそうなるのか、理由は環境によって異なります。

**EF / EF Core 環境の場合：** DbContext自体が変更追跡・Identity Mapを内包しているため、別途UoWクラスを作るならトランザクション境界の管理のみが残ります。

**Dapper 環境の場合：** 変更追跡とIdentity Mapは「ORMがSQLを自動生成するから必要になる機能」です。EFでは開発者がオブジェクトを操作するだけでSQLを書かないため、EFが「どのオブジェクトが変わったか」を追跡してUPDATE文を生成する必要があります。Dapperは違います。開発者が自分でSQLを書くため、何が変わったかはSQL自体に表現されています。追跡する問題が最初から発生しない。

ここで「ADO.NETも同じでは？」という疑問が生まれます。ADO.NETにも変更追跡・Identity Mapはありません。違いは**意図**です。Part 1のADO.NET時代の開発者はこれらの機能を**欲しかった**。だからUoW自身が手動で変更追跡・Identity Mapを実装しようとし、それが「古典的UoW」が巨大化した理由でした。一方Dapper採用者はEFのオーバーヘッドを嫌って**意図的に**その層を手放しています。「機能がない」のではなく「不要だからそのツールを選んだ」という設計判断の差です。

つまりDapper環境での「変更追跡・Identity Mapがない」は「機能が欠けている」ではなく、「その機能が解決しようとしている問題が、Dapperの使い方では発生しない」が正確な理解です。結果として、UoWに残る責務はトランザクション管理のみになります。

> **注記：** 「ADO.NET時代はUoWに変更追跡を求めた、Dapper採用者は求めない」という対比を単一の文献で示すソースは確認できていません。上記はFowler『PoEAA』の設計意図とDapperのREADMEに示された採用目的から導出した筆者の解釈です。

以降では、Dapper・生SQL環境向けの2つのパターンを解説します。

## パターン A：ベーシックUoW（Dapper・生SQL環境）

### 時代背景（2015年〜2020年頃）

**技術環境**  

- .NET Core の登場（2016年）：クロスプラットフォーム、モジュール化
- マイクロサービスアーキテクチャの台頭
- **軽量ORM（Dapper）の人気上昇**：EF Coreのオーバーヘッドを避けたいケースで採用増加
- ASP.NET Core の組み込みDIコンテナが標準に

**言語の進化（C# 6.0〜8.0）**  

- C# 6.0（2015年）：null条件演算子（`?.`）、文字列補間（`$""`）、式形式メンバー
- C# 7.0（2017年）：タプル、パターンマッチング、ローカル関数
- C# 8.0（2019年）：`IAsyncDisposable`、null許容参照型、`await using`

### このパターンが生まれた背景

ASP.NET Core の組み込みDIコンテナにより、**リポジトリを個別にDI登録し、UoWはトランザクション管理に専念する**設計が可能になりました。Part 1の「古典的UoW」では不可能だった疎結合が、DIコンテナの成熟で実現されました。

```txt
Part 1：UoWがリポジトリを生成・管理
↓
Part 3, パターンA：DIコンテナがすべてを管理

     [ DI Container (Request Scope) ]
        ↙            ↓          ↘
[ Unit of Work ] [ Repo A ] [ Repo B ]
        ↘            ↓          ↙
       [ Shared DB Connection ]
```

### Session / Connection 管理パターン

接続とトランザクションの状態を `IDbSession` として分離し、リポジトリとUoWが共有する設計です。

```csharp
// セッション管理（状態保持のみ）
public interface IDbSession
{
    IDbConnection Connection { get; }
    IDbTransaction Transaction { get; }
}

public interface IDbSessionManager : IDbSession
{
    new IDbTransaction Transaction { get; set; }
}

public class DbSession : IDbSessionManager, IDisposable
{
    private readonly IDbConnection _connection;

    public DbSession(IDbConnection connection)
    {
        _connection = connection;
    }

    public IDbConnection Connection
    {
        get { return _connection; }
    }

    public IDbTransaction Transaction { get; set; }

    public void Dispose()
    {
        if (Transaction != null)
        {
            Transaction.Dispose();
        }
        if (_connection != null)
        {
            _connection.Dispose();
        }
    }
}
```

責務の分離ポイント：  

- `IDbSession`（読み取り専用）→ リポジトリに注入。誤ってTransactionを書き換えられない
- `IDbSessionManager`（読み書き可能）→ UoWに注入。トランザクションをセットできる

### UoW実装

```csharp
public interface IUnitOfWork : IDisposable, IAsyncDisposable
{
    Task BeginTransactionAsync(CancellationToken ct = default);
    Task CommitAsync(CancellationToken ct = default);
    Task RollbackAsync(CancellationToken ct = default);
}

public class UnitOfWork : IUnitOfWork
{
    private readonly IDbSessionManager _sessionManager;
    private bool _disposed;

    public UnitOfWork(IDbSessionManager sessionManager)
    {
        _sessionManager = sessionManager;
    }

    public async Task BeginTransactionAsync(CancellationToken ct = default)
    {
        if (_sessionManager.Transaction != null)
        {
            throw new InvalidOperationException("Transaction already started");
        }

        if (_sessionManager.Connection.State != ConnectionState.Open)
        {
            await ((DbConnection)_sessionManager.Connection).OpenAsync(ct);
        }

        _sessionManager.Transaction = await ((DbConnection)_sessionManager.Connection)
            .BeginTransactionAsync(ct);
    }

    public async Task CommitAsync(CancellationToken ct = default)
    {
        if (_sessionManager.Transaction == null)
        {
            throw new InvalidOperationException("No active transaction");
        }

        try
        {
            await ((DbTransaction)_sessionManager.Transaction).CommitAsync(ct);
        }
        finally
        {
            await ((DbTransaction)_sessionManager.Transaction).DisposeAsync();
            _sessionManager.Transaction = null;
        }
    }

    public async Task RollbackAsync(CancellationToken ct = default)
    {
        if (_sessionManager.Transaction == null)
        {
            return;
        }

        try
        {
            await ((DbTransaction)_sessionManager.Transaction).RollbackAsync(ct);
        }
        finally
        {
            await ((DbTransaction)_sessionManager.Transaction).DisposeAsync();
            _sessionManager.Transaction = null;
        }
    }

    public void Dispose()
    {
        if (_disposed) return;
        _sessionManager.Transaction?.Dispose();
        _disposed = true;
    }

    public async ValueTask DisposeAsync()
    {
        if (_disposed) return;
        if (_sessionManager.Transaction is DbTransaction t)
        {
            await t.DisposeAsync();
        }
        _disposed = true;
    }
}
```

### リポジトリ側

```csharp
public class OrderRepository : IOrderRepository
{
    private readonly IDbSession _session;

    public OrderRepository(IDbSession session)
    {
        _session = session;
    }

    public async Task<int> CreateAsync(Order order)
    {
        const string sql = @"
            INSERT INTO Orders (CustomerId, OrderDate, TotalAmount)
            VALUES (@CustomerId, @OrderDate, @TotalAmount);
            SELECT CAST(SCOPE_IDENTITY() as int);";

        return await _session.Connection.ExecuteScalarAsync<int>(
            sql, order, _session.Transaction);
    }

    public async Task<Order> GetByIdAsync(int id)
    {
        const string sql = "SELECT * FROM Orders WHERE Id = @Id";
        return await _session.Connection.QueryFirstOrDefaultAsync<Order>(
            sql, new { Id = id }, _session.Transaction);
    }
}
```

### DI登録

```csharp
// Startup.cs / Program.cs
services.AddScoped<IDbConnection>(_ =>
    new SqlConnection(connectionString));
services.AddScoped<DbSession>();
services.AddScoped<IDbSession>(sp => sp.GetRequiredService<DbSession>());
services.AddScoped<IDbSessionManager>(sp => sp.GetRequiredService<DbSession>());
services.AddScoped<IUnitOfWork, UnitOfWork>();
services.AddScoped<IOrderRepository, OrderRepository>();
services.AddScoped<IInventoryRepository, InventoryRepository>();
```

### サービス層での使用

```csharp
public class OrderService : IOrderService
{
    private readonly IUnitOfWork _uow;
    private readonly IOrderRepository _orderRepo;
    private readonly IInventoryRepository _inventoryRepo;

    public OrderService(
        IUnitOfWork uow,
        IOrderRepository orderRepo,
        IInventoryRepository inventoryRepo)
    {
        _uow = uow;
        _orderRepo = orderRepo;
        _inventoryRepo = inventoryRepo;
    }

    public async Task<int> CreateOrderAsync(Order order)
    {
        try
        {
            await _uow.BeginTransactionAsync();

            var product = await _inventoryRepo.GetByIdAsync(order.ProductId);
            if (product == null)
            {
                throw new NotFoundException($"Product not found: {order.ProductId}");
            }

            if (product.Stock < order.Quantity)
            {
                throw new BusinessException("Insufficient stock");
            }

            var orderId = await _orderRepo.CreateAsync(order);
            await _inventoryRepo.UpdateStockAsync(
                order.ProductId, product.Stock - order.Quantity);

            await _uow.CommitAsync();
            return orderId;
        }
        catch
        {
            await _uow.RollbackAsync();
            throw;
        }
    }
}
```

### このパターンの評価

**メリット：**  

- UoWはトランザクション管理のみ（単一責任の原則）
- UoWとリポジトリが疎結合（DIコンテナが管理）
- ORMに依存しない（Dapper、ADO.NET、どれでも使える）
- テストが容易（リポジトリとUoWを個別にモック可能）
- 理解しやすく、既存チームへの導入ハードルが低い

**デメリット（次のパターンで解消する）：**  

- `try-catch-rollback` の定型文がサービス層に散らばる
- 「在庫不足」のような予期されたビジネスルール違反も例外で表現するしかない
- 呼び出し元がどんな失敗パターンがあるかをシグネチャから読み取れない

**この設計を可能にした技術：**  

1. **ASP.NET Core の組み込みDIコンテナ**：リポジトリの個別登録と自動注入。Part 1では不可能だった責務の分離を実現
2. **リクエストスコープ管理**：1リクエスト1インスタンスが保証され、接続とトランザクションを安全に共有できる
3. **`IAsyncDisposable`（C# 8.0）**：`await using` による非同期リソース解放が標準化された

この「デメリット」を解消するために生まれたのが次のスコープベースUoWです。

## パターン B：スコープベースUoW（Result型 + 自動トランザクション判定）

### このパターンの立ち位置

パターンAはオーソドックスな実装ですが、サービスメソッドごとに try-catch-rollback のボイラープレートが繰り返されます。「ビジネスロジックをデリゲートとして渡し、トランザクション制御を委譲すれば解決できるのでは」という発想から生まれたのがこのパターンです。

一方、さらに先へ進む選択肢として、のちに解説するパターンC（MediatR + Pipeline Behaviorベース）があります。ただしパターンCへの移行には Minimal API や MediatR の導入が必要で、コントローラーベースのレガシー環境では破壊的変更になります。

パターンBはその**中間点**です。コントローラーベース・Dapper・EF Core不使用というレガシー環境の制約を守りながら、Result型とラムダスコープだけでボイラープレートを排除できる、導入可能なギリギリのラインとして設計しました。

> **このパターンは、上記項目をもとに発想し、Claude（Anthropic）との対話を通じてコードに落とし込んだものです。既存の文献に直接の出典を持たない実践的アプローチです。アイデアの出発点は筆者ですが、実装の具体化はAIとの共同作業によるものです。**

### 時代背景（2020年〜現在）

**技術環境**  

- .NET 5〜現在：統一プラットフォーム、パフォーマンス大幅改善
- **関数型プログラミングの影響**：F#、Rust、HaskellのアイデアがC#に浸透
- Railway Oriented Programming の概念が普及
- Result型ライブラリの登場・成熟：FluentResults、LanguageExt、ErrorOr

**言語の進化（C# 8.0〜現在）**  

- C# 8.0（2019年）：null許容参照型、`await using`、パターンマッチング強化
- C# 9.0（2020年）：レコード型、`init`-only プロパティ
- C# 10.0（2021年）：グローバルusing、ファイルスコープ名前空間
- C# 11.0（2022年）：`required` メンバー、生文字列リテラル
- C# 12.0（2023年）：**プライマリコンストラクタ**（このパターンの実装で使用）

**設計思想の変化**  

- 「例外は例外的な状況のみ」という原則の再評価
- 例外をフロー制御に使わない設計への移行
- エラーハンドリングを型システムで表現する方向性
- 不変性（Immutability）の重視

### 解決したかった2つの問題

**問題1：ボイラープレートの散乱**  

パターンAのサービス層には、毎回このコードが必要です。

```csharp
// ❌ すべてのサービスメソッドに同じ定型文が必要
try
{
    await _uow.BeginTransactionAsync();
    // ビジネスロジック
    await _uow.CommitAsync();
}
catch
{
    await _uow.RollbackAsync();
    throw;
}
```

**問題2：「例外による処理中断」しか手段がない**  

「在庫不足」のような**予期されたビジネスルール違反**を処理したいとき、例外をスローするしか方法がありません。

```csharp
// ❌ 予期されたビジネスルール違反を例外で表現するのは概念的に歪んでいる
if (product.Stock < order.Quantity)
    throw new BusinessException("Insufficient stock");
```

例外は本来、「予期しないエラー」のためのものです。バリデーション違反や在庫不足は「予期された失敗」であり、例外で表現するのは適切ではありません。また、`Task<int>` というシグネチャを見ても、呼び出し側はどんな失敗パターンがあるか知ることができません。

### Result型による解決

F# や Rust の影響を受けた **Result型** を使うことで、「成功」と「失敗」を型として明示できます。

```csharp
// ✅ シグネチャからエラーの可能性が一目でわかる
public Task<Result<int>> CreateOrderAsync(Order order)
```

`Task<int>` を返す場合、呼び出し側はどんなエラーが起きるか知りません。`Task<Result<int>>` を返す場合、「このメソッドは失敗する可能性がある」とシグネチャで明示されます。

### スコープベースUoW：ラムダでトランザクション境界を包む

> **実装について：** 以降のコード例は **FluentResults** を使った場合で実装を進めます。他のResult型ライブラリへの差し替え方法は後述の「[Result型ライブラリについて](#result型ライブラリについて)」を参照してください。
<!--  -->
> **パターンAとの関係：** このパターンはパターンAの `IDbSession` / `IDbSessionManager` をそのまま流用します。DI登録・Session管理の仕組みはパターンAと共通であり、UoWクラスの実装だけが異なります。リポジトリへのトランザクション受け渡しも `IDbSession` 経由で行います。

ボイラープレートを排除するため、**トランザクションの境界をラムダ式でカプセル化**し、Result型の状態でコミット/ロールバックを自動判定します。また、ネストされた呼び出しを設計違反として即座に検出する仕組みを持ちます。

```csharp
public interface IUnitOfWork : IDisposable, IAsyncDisposable
{
    // ラムダ式でビジネスロジックを渡すだけ
    Task<Result<T>> ExecuteInTransactionAsync<T>(
        Func<Task<Result<T>>> operation,
        CancellationToken ct = default);
}

// IDbSessionManager はパターンAで定義済みのものを流用
public class UnitOfWork(
    IDbSessionManager sessionManager,
    ILogger<UnitOfWork> logger) : IUnitOfWork
{
    // ネストされたトランザクション検出用
    // AsyncLocal は async/await 境界を跨いでも正確にスコープを追跡できる
    // 同一HTTPリクエスト内でのみ値が共有され、並行リクエストとは分離される
    private static readonly AsyncLocal<bool> IsInTransaction = new();

    public async Task<Result<T>> ExecuteInTransactionAsync<T>(
        Func<Task<Result<T>>> operation,
        CancellationToken ct = default)
    {
        // サービスAがサービスBを呼び出し、両方がこのメソッドを呼ぶケースを検出する
        if (IsInTransaction.Value)
            throw new InvalidOperationException(
                "Nested transaction detected. This indicates a design issue: " +
                "a service is calling another service that also uses UoW.");

        IsInTransaction.Value = true;

        try
        {
            if (sessionManager.Connection.State != ConnectionState.Open)
                await ((DbConnection)sessionManager.Connection).OpenAsync(ct);

            var transaction = await ((DbConnection)sessionManager.Connection)
                .BeginTransactionAsync(ct);

            // セッションにトランザクションをセット → リポジトリが IDbSession 経由で参照できる
            sessionManager.Transaction = transaction;

            logger.LogDebug("Transaction started");

            var result = await operation();  // ← ラムダの中だけビジネスロジック

            if (result.IsSuccess)
            {
                await ((DbTransaction)transaction).CommitAsync(ct);
                logger.LogInformation("Transaction committed");
            }
            else
            {
                await ((DbTransaction)transaction).RollbackAsync(ct);
                logger.LogWarning("Transaction rolled back: {Errors}",
                    string.Join(", ", result.Errors));
            }

            return result;
        }
        catch (Exception ex)
        {
            if (sessionManager.Transaction is DbTransaction t)
                await t.RollbackAsync(ct);
            logger.LogError(ex, "Transaction rolled back due to exception");
            throw;
        }
        finally
        {
            if (sessionManager.Transaction is DbTransaction t)
                await t.DisposeAsync();
            sessionManager.Transaction = null;
            IsInTransaction.Value = false;  // スコープ終了
        }
    }

    public void Dispose() { }
    public ValueTask DisposeAsync() => default;
}
```

**パターンAとの差分はここだけです。** `BeginTransactionAsync` / `CommitAsync` / `RollbackAsync` の呼び出しをラムダの内側に閉じ込め、Result型の成否でコミット/ロールバックを自動判定します。リポジトリは引き続き `IDbSession.Transaction` を参照するため、変更は不要です。

**ポイント：**  

- `Result.IsSuccess` が `true` → コミット
- `Result.IsSuccess` が `false` → ロールバック（**例外なし**）
- 例外 → ロールバック（従来通り）

`return Result.Fail<int>("在庫不足")` を返した時点で、ロールバックが自動実行されます。例外スローは不要です。

**ネストされたトランザクションの検出**  

サービスAがサービスBを呼び出し、両方が `ExecuteInTransactionAsync` を呼ぶケースがあります。内側の呼び出しが外側のトランザクションを上書きし、意図しないロールバックが発生します。`AsyncLocal<bool>` はこれを即座に検出して例外を投げます。サイレントに無視したり自動的にネストを許可したりするより、早期に検出して原因を特定する方が安全です。

### サービス層での使用

```csharp
public class OrderService(
    IUnitOfWork uow,
    IOrderRepository orderRepo,
    IInventoryRepository inventoryRepo) : IOrderService
{
    public async Task<Result<int>> CreateOrderAsync(Order order)
    {
        // try-catch-rollbackのボイラープレートは不要
        return await _uow.ExecuteInTransactionAsync(async () =>
        {
            if (order.Items.Count == 0)
                return Result.Fail<int>("Order must have at least one item");

            var product = await _inventoryRepo.GetByIdAsync(order.ProductId);
            if (product == null)
                return Result.Fail<int>($"Product {order.ProductId} not found");

            if (product.Stock < order.Quantity)
                return Result.Fail<int>(
                    $"Insufficient stock. Available: {product.Stock}, " +
                    $"Requested: {order.Quantity}");

            var orderId = await _orderRepo.CreateAsync(order);
            await _inventoryRepo.UpdateStockAsync(
                order.ProductId, product.Stock - order.Quantity);

            return Result.Ok(orderId);
        });
    }
}
```

### コントローラー層での使用

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;

    public OrdersController(IOrderService orderService)
    {
        _orderService = orderService;
    }

    [HttpPost]
    public async Task<IActionResult> CreateOrder([FromBody] CreateOrderRequest request)
    {
        var result = await _orderService.CreateOrderAsync(request.ToOrder());

        if (result.IsSuccess)
            return Ok(new { orderId = result.Value });

        return result.Errors.First() switch
        {
            var e when e.Metadata.TryGetValue("ErrorType", out var et) && et?.ToString() == "NotFound"
                => NotFound(new { errors = result.Errors }),
            var e when e.Metadata.TryGetValue("ErrorType", out var et) && et?.ToString() == "Validation"
                => BadRequest(new { errors = result.Errors }),
            _ => StatusCode(500, new { errors = result.Errors })
        };
    }
}
```

### Result型ライブラリについて

本記事のコード例は **FluentResults** を前提として実装しています。シンプルなAPIと豊富なドキュメントから、Result型を導入する際の最初の選択肢として最もポピュラーなライブラリです。

```csharp
// NuGet: FluentResults
// using FluentResults;

return Result.Fail<int>("Insufficient stock");   // 失敗
return Result.Ok(orderId);                       // 成功
```

ただし `ExecuteInTransactionAsync` が判定するのは `result.IsSuccess` という概念だけです。FluentResultsに限らず、同等のインターフェースを持つライブラリであれば差し替えて使用できます。

#### ErrorOr

近年人気上昇中。エラーの種別（Validation / NotFound / Conflictなど）を型として表現できる。HTTPレスポンスへのマッピングが直感的で、ASP.NET Core Minimal APIとの相性が良い。

```csharp
// using ErrorOr;
return Error.Validation(description: "Order must have items");
return Error.NotFound(description: $"Product {order.ProductId} not found");
return orderId; // 暗黙的にErrorOr<int>.Successに変換
```

#### LanguageExt

最も関数型プログラミングに忠実なライブラリ。`Either<L, R>` 型でRailway Oriented Programmingを厳密に実践できる。個人開発や小規模なサンプルでは過剰に感じることもあるが、DDDを徹底したエンタープライズ開発では最も厳密な設計が可能。

> **注意：** LanguageExtを使う場合、`IUnitOfWork` のインターフェース定義自体を `Task<Either<Error, T>>` を返す形に変更する必要があります。本記事の `Task<Result<T>>` ベースのインターフェースとは型が合わないため、そのままでは差し替えできません。

```csharp
// using LanguageExt;
public async Task<Either<Error, int>> CreateOrderAsync(Order order)
{
    return await _uow.ExecuteInTransactionAsync(async () =>
    {
        Either<Error, Order> validated = ValidateOrder(order);
        return await validated.BindAsync(async o => await _orderRepo.CreateAsync(o));
    });
}
```

#### CSharpFunctionalExtensions

メソッドチェーンスタイルでRailway Oriented Programmingを実践できる。FluentResultsより関数型寄り、LanguageExtより学習コストが低い。段階的な関数型導入に向いている。

```csharp
return await Result.Success(order)
    .Ensure(o => o.Items.Any(), "Order must have items")
    .Bind(async o => await _orderRepo.CreateAsync(o));
```

どのライブラリを選ぶかは、チームの関数型プログラミングへの習熟度と、エラー表現の厳密さをどこまで求めるかで判断してください。

### ベーシックUoWとの比較

| 項目 | ベーシックUoW（パターンA） | スコープベースUoW（パターンB） |
| ------ | -------------------------- | ------------------------------ |
| トランザクション制御 | 手動で Begin/Commit/Rollback | ラムダ内で自動化 |
| エラーハンドリング | 例外（try-catch必須） | Result型（型安全） |
| ボイラープレート | 毎回 try-catch-rollback | 完全排除 |
| シグネチャの表現力 | エラーの可能性が不明 | シグネチャで明示 |
| 学習コスト | 低い | 中程度（Result型の理解が必要） |
| 既存コードへの段階導入 | 容易 | やや困難 |

### このパターンの評価

**メリット：**  

- `try-catch` が不要：エラーハンドリングが型で表現される
- シグネチャから振る舞いが予測可能：`Task<Result<T>>` を見れば失敗する可能性があると分かる
- トランザクションとエラー処理が自動判定：Result の状態でコミット/ロールバックが決まる
- 制御フローの明晰さ：例外による非局所的なジャンプがなく、デバッグしやすい
- 関数型ライブラリとの親和性：Map、Bind、Matchなどの操作と自然につながる

**デメリット：**  

- Result型・Railway Oriented Programmingの理解が前提
- 既存コードへの段階導入が難しい場合がある
- チーム全体での合意が必要：一部のメンバーだけでは効果が薄い
- 設計志向の強いプロジェクト向けであり、万能ではない

**この設計を可能にした技術：**  

1. **ジェネリクス**：`Result<T>` のような型安全な設計が可能に
2. **ラムダ式とクロージャ**：`ExecuteInTransactionAsync(async () => { ... })` という宣言的な記述
3. **async/await**：非同期処理の自然な記述と `AsyncLocal` によるスコープ追跡
4. **Nullable Reference Types（C# 8.0）**：nullに関するエラーを型システムで防ぐ
5. **関数型ライブラリの普及**：FluentResults、LanguageExt、ErrorOr など

**このパターンが特に有効な環境：**  

- Dapper や ADO.NET を使用している
- コントローラーベースの ASP.NET Core
- ビジネスルールが複雑で、複数の失敗パターンがある
- 型安全なエラーハンドリングを重視したい
- try-catch-rollbackの繰り返しを解消したい

> **位置づけについて：** パターンBは「設計品質を重視するプロジェクト向けの選択肢」です。中規模業務アプリケーションの標準的な実装ではなく、Result型と関数型的なエラーハンドリングに習熟したチームが採用する、設計志向の強いアプローチです。パターンAで十分な場面は多くあります。

### コラム：Result型を使うと、例外は不要になるのか？

不要にはなりません。Result型と例外は役割が異なります。

「在庫不足」「バリデーション失敗」のような**予期されたビジネスルール違反**にはResult型を使います。これらは「起こりうる正常なケース」であり、例外は本来この用途には向いていません。一方、「DB接続断」「タイムアウト」「予期しないnull参照」のような**予期しないシステムエラー**には引き続き例外を使います。

`ExecuteInTransactionAsync` はどちらのケースもロールバックします。Result型の失敗（`result.IsSuccess == false`）でも、例外の発生でも、トランザクションはロールバックされます。

### コラム：パターンAとBは共存できるか？

できます。同じ `IDbSession` / `IDbSessionManager` を共有しているため、DI登録の仕組みはそのままで、コマンドごとに使い分けが可能です。

既存のコードをパターンAで維持しながら、新しい機能からパターンBを段階的に導入するのが現実的なアプローチです。Result型に不慣れなメンバーはパターンA、慣れたメンバーはパターンBを使うという混在も許容できます。

## パターン C：パイプラインUoW（VSA + MediatR 環境）

### 時代背景（2023年〜現在）

**技術環境**  

- .NET 6〜現在：Minimal API の成熟
- **Vertical Slice Architecture（VSA）の台頭**
- **MediatR Pipeline Behaviors の普及**
- Carter、FastEndpoints などの軽量エンドポイントライブラリ

**言語の進化（C# 10.0〜）**  

- C# 10.0（2021年）：グローバルusing、ファイルスコープ名前空間
- C# 11.0（2022年）：required メンバー
- C# 12.0（2023年）：**プライマリコンストラクタ**、Collection Expressions

### 概要

VSAとMediatRを採用した場合、**トランザクション管理をPipeline Behaviorに委ねる**ことで、ハンドラーがビジネスロジックのみに集中できます。

Pipeline Behavior 自体がUoWの役割を担うため、UoWクラスの実装自体が存在しません。

```txt
[ HTTP Request ]
       ↓
[ Minimal API Endpoint ]
       ↓
[ MediatR.Send(Command) ]
       ↓
┌───────────────────────────────────────┐
│ Pipeline Behaviors（横断的処理）       │
│  1. LoggingBehavior                   │
│  2. ValidationBehavior                │
│  3. TransactionBehavior ← これがUoW   │
└───────────────────────────────────────┘
       ↓
[ Handler（ビジネスロジックのみ）]
```

### なぜこの設計が生まれたのか：VSA とレイヤードアーキテクチャの問題

パターンCはVertical Slice Architecture（VSA）を前提とします。VSAはMediatR・AutoMapperの作者Jimmy Bogardが提唱したアーキテクチャで、従来のレイヤードアーキテクチャが抱える「1つの機能追加で複数のレイヤーを横断的に変更しなければならない」という問題を解決します。VSAでは機能単位で垂直にスライスし、1つのユースケースに関するすべてのコード（エンドポイント・ハンドラー・バリデーション・データアクセス）を1つのフォルダに集約します。

**プロジェクト構成（VSAの典型例）**  

```txt
src/
├── Features/                    # 機能単位のスライス
│   ├── Orders/
│   │   ├── CreateOrder/
│   │   │   ├── CreateOrderCommand.cs
│   │   │   ├── CreateOrderHandler.cs
│   │   │   ├── CreateOrderValidator.cs
│   │   │   └── CreateOrderEndpoint.cs
│   │   ├── GetOrderById/
│   │   │   ├── GetOrderByIdQuery.cs
│   │   │   ├── GetOrderByIdHandler.cs
│   │   │   └── GetOrderByIdEndpoint.cs
│   │   └── CancelOrder/
│   │       ├── CancelOrderCommand.cs
│   │       ├── CancelOrderHandler.cs
│   │       └── CancelOrderEndpoint.cs
│   └── Products/
│       └── ...
├── Common/
│   ├── Behaviors/               # Pipeline Behaviors（横断的処理）
│   │   ├── TransactionBehavior.cs  ← これがUoWの役割
│   │   ├── ValidationBehavior.cs
│   │   └── LoggingBehavior.cs
│   ├── Database/
│   │   └── ApplicationDbContext.cs
│   └── Abstractions/
│       ├── ICommand.cs          ← 書き込み操作（トランザクションあり）
│       ├── IQuery.cs            ← 読み取り操作（トランザクションなし）
│       └── Result.cs
└── Program.cs
```

### コマンド・クエリの抽象化（CQRS）

VSA + MediatRでは、書き込み操作（Command）と読み取り操作（Query）を型で区別します。この区別が **TransactionBehaviorをCommandのみに適用する** 根拠になります。

素のMediatRでは全操作が `IRequest<TResponse>` を実装するため、TransactionBehaviorの型制約 `where TRequest : ICommand` が書けず、クエリにまでトランザクションがかかってしまいます。マーカーインターフェースを挟むことで、型レベルでCommandとQueryを分離します。

```csharp
// 書き込み操作：ICommand を実装 → TransactionBehavior が自動適用される
// IRequest<Result> を継承することで MediatR の IRequestHandler と繋がる
public interface ICommand : IRequest<Result> { }
public interface ICommand<TResponse> : IRequest<Result<TResponse>> { }

// 読み取り操作：IQuery を実装 → where TRequest : ICommand を満たさないため
// TransactionBehavior はスキップされる（トランザクションなし）
public interface IQuery<TResponse> : IRequest<Result<TResponse>> { }
```

ハンドラー側の定義は素のMediatRと変わりません。`CreateOrderCommand : ICommand<int>` が `IRequest<Result<int>>` を満たすため、`IRequestHandler<CreateOrderCommand, Result<int>>` というハンドラー定義がそのまま成立します。

### TransactionBehavior の実装

#### 標準：EF Core版

パイプライン構成を採用するプロジェクトであれば、EF Coreを使っていることがほとんどです。EF Core版が実質的な標準実装です。`CreateExecutionStrategy()` による自動リトライ対応と、`SaveChangesAsync()` をここで呼ぶことでハンドラーをビジネスロジックに専念させる点がポイントです。

```csharp
// where TRequest : ICommand　← Commandのみに適用。IQueryにはトランザクションを張らない
public class TransactionBehavior<TRequest, TResponse>(
    ApplicationDbContext dbContext,
    ILogger<TransactionBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : ICommand
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var strategy = dbContext.Database.CreateExecutionStrategy();

        return await strategy.ExecuteAsync(async () =>
        {
            await using var transaction = await dbContext.Database
                .BeginTransactionAsync(cancellationToken);

            try
            {
                var response = await next();

                if (response is Result result && result.IsSuccess)
                {
                    await dbContext.SaveChangesAsync(cancellationToken);
                    await transaction.CommitAsync(cancellationToken);
                    logger.LogInformation(
                        "Transaction committed for {Request}",
                        typeof(TRequest).Name);
                }
                else
                {
                    await transaction.RollbackAsync(cancellationToken);
                }

                return response;
            }
            catch (Exception ex)
            {
                await transaction.RollbackAsync(cancellationToken);
                logger.LogError(ex,
                    "Transaction rolled back for {Request}",
                    typeof(TRequest).Name);
                throw;
            }
        });
    }
}
```

#### 参考：IDbConnection版（Dapper / 生SQL環境）

EF Coreを使わない環境向けの実装です。実質的には、パターンAやBで扱ってきたトランザクション管理がPipeline Behaviorに移動しただけです。構造の本質は同じです。

```csharp
// where TRequest : ICommand　← Commandのみに適用。IQueryにはトランザクションを張らない
public class TransactionBehavior<TRequest, TResponse>(
    IDbConnection connection,
    ILogger<TransactionBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : ICommand
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (connection.State != ConnectionState.Open)
            await ((DbConnection)connection).OpenAsync(cancellationToken);

        await using var transaction =
            await ((DbConnection)connection).BeginTransactionAsync(cancellationToken);

        try
        {
            var response = await next();

            if (response is Result result && result.IsSuccess)
                await ((DbTransaction)transaction).CommitAsync(cancellationToken);
            else
                await ((DbTransaction)transaction).RollbackAsync(cancellationToken);

            return response;
        }
        catch (Exception ex)
        {
            await ((DbTransaction)transaction).RollbackAsync(cancellationToken);
            logger.LogError(ex,
                "Transaction rolled back for {Request}", typeof(TRequest).Name);
            throw;
        }
    }
}
```

### ハンドラーの実装例

**Commandハンドラー（書き込み）：TransactionBehaviorが自動適用**  

```csharp
// ICommand を実装 → TransactionBehavior が自動でトランザクションを張る
public record CreateOrderCommand(int CustomerId, List<OrderItem> Items)
    : ICommand<int>;

// トランザクションを一切意識しない
public class CreateOrderHandler(
    ApplicationDbContext dbContext,
    ILogger<CreateOrderHandler> logger)
    : IRequestHandler<CreateOrderCommand, Result<int>>
{
    public async Task<Result<int>> Handle(
        CreateOrderCommand request,
        CancellationToken cancellationToken)
    {
        var customer = await dbContext.Customers
            .FindAsync([request.CustomerId], cancellationToken);

        if (customer == null)
            return Result.Fail<int>($"Customer {request.CustomerId} not found");

        foreach (var item in request.Items)
        {
            var product = await dbContext.Products
                .FindAsync([item.ProductId], cancellationToken);

            if (product == null)
                return Result.Fail<int>($"Product {item.ProductId} not found");

            if (product.Stock < item.Quantity)
                return Result.Fail<int>(
                    $"Insufficient stock for {product.Name}");

            product.Stock -= item.Quantity;
        }

        var order = new Order
        {
            CustomerId = request.CustomerId,
            OrderDate = DateTime.UtcNow,
            Items = request.Items.Select(i => new OrderItem
            {
                ProductId = i.ProductId,
                Quantity = i.Quantity,
                UnitPrice = i.Price
            }).ToList()
        };

        dbContext.Orders.Add(order);

        // DB生成IDが必要な場合はここで SaveChangesAsync を呼ぶ
        // TransactionBehavior がトランザクションを保持しているためコミットはしない
        // EF Core が INSERT 後に order.Id へ DB生成値を自動で書き戻す
        await dbContext.SaveChangesAsync(ct);

        return Result.Ok(order.Id);
        // TransactionBehavior が Result.IsSuccess を確認後、SaveChanges → Commit を実行
    }
}
```

**Queryハンドラー（読み取り）：TransactionBehaviorは適用されない**  

```csharp
// IQuery を実装 → where TRequest : ICommand の制約を満たさないため
// TransactionBehavior はスキップされる（トランザクションなし）
public record GetOrderByIdQuery(int OrderId) : IQuery<OrderDto>;

public class GetOrderByIdHandler(ApplicationDbContext dbContext)
    : IRequestHandler<GetOrderByIdQuery, Result<OrderDto>>
{
    public async Task<Result<OrderDto>> Handle(
        GetOrderByIdQuery request,
        CancellationToken cancellationToken)
    {
        // 読み取りのみ。トランザクションは不要
        var order = await dbContext.Orders
            .AsNoTracking() // 変更追跡も不要
            .Where(o => o.Id == request.OrderId)
            .Select(o => new OrderDto(o.Id, o.CustomerId, o.OrderDate))
            .FirstOrDefaultAsync(cancellationToken);

        if (order is null)
            return Result.Fail<OrderDto>($"Order {request.OrderId} not found");

        return Result.Ok(order);
    }
}
```

### MediatR 登録

```csharp
builder.Services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(Program).Assembly);

    // Pipeline Behaviors の登録（順序が重要）
    cfg.AddOpenBehavior(typeof(LoggingBehavior<,>));
    cfg.AddOpenBehavior(typeof(ValidationBehavior<,>));
    cfg.AddOpenBehavior(typeof(TransactionBehavior<,>));
    // TransactionBehavior は ICommand のみに適用される（IQuery はスキップ）
});
```

### Pipeline の実行順序

リクエストがどのような順序で処理されるかを示します。Pipeline Behaviorはネスト構造になっており、リクエスト時とレスポンス時の両方で処理を挟み込みます。

```txt
HTTP Request
    ↓
Minimal API Endpoint
    ↓
MediatR.Send(command)
    ↓
┌─────────────────────────────────┐
│ 1. LoggingBehavior              │ ← リクエストをログ
└─────────────────────────────────┘
    ↓
┌─────────────────────────────────┐
│ 2. ValidationBehavior           │ ← バリデーション実行（FluentValidation）
└─────────────────────────────────┘
    ↓
┌─────────────────────────────────┐
│ 3. TransactionBehavior          │ ← トランザクション開始
└─────────────────────────────────┘
    ↓
┌─────────────────────────────────┐
│ 4. Handler (ビジネスロジック)    │ ← 純粋なビジネスロジック（SaveChanges不要）
└─────────────────────────────────┘
    ↓
TransactionBehavior が Result判定 → SaveChanges → Commit / Rollback
    ↓
ValidationBehavior 完了
    ↓
LoggingBehavior がレスポンスをログ
    ↓
HTTP Response
```

**ポイント：** TransactionBehaviorはHandlerを「外側から包む」構造です。`next()` を呼び出してHandlerを実行し、その結果を受け取ってからコミット/ロールバックを判定します。HandlerはトランザクションもSaveChangesも意識する必要がありません。

### コラム：EF Core + パイプラインで、TransactionBehavior は本当に必要か？

パイプライン方式で EF Core を使うとき、こんな疑問が生まれます。

> 「EF Core の `SaveChangesAsync` が内部でトランザクションを張るなら、TransactionBehavior はいらないのでは？」

**その通りです。** EF Core だけで完結する純粋な環境では、以下のように `SaveChangesAsync` 1回で十分です。

```txt
EF Core のみの場合：
Handler が dbContext を操作
    ↓
SaveChangesAsync（TransactionBehaviorが呼ぶ）
    ↓
EF Core が内部で自動的にトランザクションを張ってコミット
→ TransactionBehavior は「あっても害はないが、なくても動く」
```

**では、なぜ TransactionBehavior を持つのか**  

現実のエンタープライズ開発では、「EF Core だけで完結する」プロジェクトの方が少数派です。TransactionBehavior が必要になる主なケースは2つです。

- **ケース1：`SaveChangesAsync` を複数回呼ぶ必要がある**  
  EF Core のデフォルト動作では `SaveChangesAsync` ごとに独立したトランザクションが張られます。DB生成IDを中間で取得して後続処理に渡す場合など、`SaveChangesAsync` を複数回呼ぶ処理を1つのトランザクションに束ねるには、明示的なトランザクション管理が必要です。

- **ケース2：EF Core と Dapper・生SQL・ストアドが混在する**  
  レガシー資産を流用している現場では、EF Core と Dapper が同一リクエスト内で混在することがあります。Dapper はトランザクションを自動管理しないため、TransactionBehavior が両者を束ねる役割を担います。

```csharp
// ケース1・2 が重なる例
public class CreateOrderHandler(
    AppDbContext dbContext,
    IDbSession session)           // IDbSession 経由で TransactionBehavior のトランザクションを参照
    : IRequestHandler<CreateOrderCommand, Result<int>>
{
    // ハンドラー内（TransactionBehaviorのトランザクション内で動いている）
    public async Task<Result<int>> Handle(CreateOrderCommand command, CancellationToken ct)
    {
        var order = command.ToOrder(); // command からエンティティを生成

        dbContext.Orders.Add(order);

        // ケース1：IDを確定させるため SaveChanges を中間で呼ぶ
        // CurrentTransaction があるためコミットはしない
        await dbContext.SaveChangesAsync(ct); // INSERTは走る。コミットはしない
        // EF Core が INSERT 後に order.Id へ DB生成値を自動で書き戻す
        // この時点で order.Id が確定している

        // ケース2：EF Core と Dapper・生SQL・ストアドが混在する
        // 確定したIDをDapperのレガシーストアドに渡す
        await session.Connection.ExecuteAsync(
            "EXEC sp_UpdateLegacySystem @Id",
            new { Id = order.Id },       // 確定したIDを使える
            session.Transaction);        // TransactionBehavior が張ったトランザクションを共有

        return Result.Ok(order.Id);
        // TransactionBehavior が Result.IsSuccess を見てここで初めてコミット
    }
}
```

> **補足：TransactionBehavior 内での `SaveChangesAsync` の動作**
>
> EF Core は `SaveChangesAsync` を呼ぶとき、`DbContext.Database.CurrentTransaction` にアクティブなトランザクションがあるかを確認します。TransactionBehavior が `BeginTransactionAsync` で張ったトランザクションがある場合、`SaveChangesAsync` は **SQLの実行のみ行い、コミットしません。** コミット権限はトランザクションの所有者（TransactionBehavior）が持っています。
>
> **出典：**
>
> - Microsoft. *Transactions - EF Core*  
>   <https://learn.microsoft.com/en-us/ef/core/saving/transactions>
> - Milan Jovanović. *Working With Transactions In EF Core* (2023)  
>   <https://www.milanjovanovic.tech/blog/working-with-transactions-in-ef-core>

**まとめ：どう判断するか**  

| 状況 | TransactionBehavior |
| ------ | ------------------- |
| EF Core のみ・`SaveChangesAsync` 1回 | 不要（あっても害はない） |
| `SaveChangesAsync` を複数回呼ぶ | 必要 |
| EF Core + Dapper/ストアド混在 | 必要 |

「VSA + MediatR を採用しつつ、TransactionBehavior を最初から持っておく」という選択は、将来の混在環境への備えとして合理的です。UoWは実装クラスとして姿を消しても、その概念と役割は Pipeline Behavior の中に生き続けています。

### このパターンの評価

**メリット：**  

- ハンドラーは純粋にビジネスロジックのみ
- トランザクション、ログ、バリデーションが横断的に自動適用
- 新しいコマンドを追加しても自動でトランザクションが適用される
- テストが容易（ハンドラーの依存が少ない）

**デメリット・注意点：**  

- VSA・CQRS・MediatRの理解が前提
- デバッグ時に実行フローが追いにくい場合がある
- レイヤードアーキテクチャとは相性が悪い
- チーム全体の理解と合意が必要

**この設計を可能にした技術：**  

1. **MediatR Pipeline Behaviors**：横断的関心事（ログ・バリデーション・トランザクション）をハンドラーから完全分離
2. **Minimal API（.NET 6+）**：エンドポイントの簡潔な定義とVSAとの相性
3. **プライマリコンストラクタ（C# 12）**：ハンドラーの依存性注入がさらに簡潔に
4. **FluentValidation**：宣言的なバリデーションをパイプラインに組み込める

## プロジェクト別の選択ガイド

### 選択フロー

```txt
EF Core を使っているか？
├── Yes → DbContext を直接使う（UoWクラス不要）
│         VSA + MediatR を採用するならパターンCも選択肢
│
└── No（Dapper / ADO.NET）
    │
    ├── VSA + MediatR を採用するか？
    │   ├── Yes → パターン C（パイプラインUoW）
    │   └── No  → パターン A or B
    │
    └── コントローラーベースか？
        ├── シンプルに始めたい → パターン A（ベーシックUoW）
        └── Result型で型安全にしたい → パターン B（スコープベースUoW）
```

### まとめ表

| プロジェクト特性 | 推奨 | 主な理由 |
| ---------------- | ------ | --------- |
| 小規模・EF Core使用 | DbContext直接使用 | DbContextで十分、UoW不要 |
| Dapper中心・シンプルに始めたい | パターンA（ベーシックUoW） | 実績あり、チームへの導入が容易 |
| Dapper中心・Result型重視 | パターンB（スコープベースUoW） | ボイラープレート排除 + 型安全 |
| VSA + MediatR 採用済み | パターンC（パイプラインUoW） | 横断的関心事の完全自動化 |
| レガシー保守 | 既存パターンを維持 | 既存コードとの整合性 |

### 混在環境でのハイブリッド構成

エンタープライズ開発では、複数のアプローチが混在することが一般的です。

```csharp
// コマンド（書き込み）：パターンB
public class CreateOrderHandler
{
    public async Task<Result<int>> Handle(...)
    {
        return await _uow.ExecuteInTransactionAsync(async () =>
        {
            // EF Core でドメインロジック
            var orderId = await _orderRepo.CreateAsync(order);

            // レガシーストアドプロシージャも同一トランザクション内で実行
            await _dbContext.Database.ExecuteSqlRawAsync(
                "EXEC UpdateInventory @ProductId, @Quantity", ...);

            return Result.Ok(orderId);
        });
    }
}

// クエリ（読み取り）：Dapper直接（UoW不要）
public class GetOrderListHandler
{
    public async Task<Result<List<OrderDto>>> Handle(...)
    {
        var orders = await _connection.QueryAsync<OrderDto>(
            "SELECT * FROM Orders WHERE CustomerId = @CustomerId", ...);
        return Result.Ok(orders.ToList());
    }
}
```

## まとめ

Unit of Work パターンの変遷をまとめると、こうなります。

**EF以前（Part 1）**：UoWがFactory・変更追跡・Identity Map・トランザクション管理を兼ねていた。DIコンテナがない時代への合理的な適応。

**Entity Framework登場（Part 2）**：DbContextがUoWの3責務をすべて内包した。単純なケースではUoWクラスは不要に。Generic Repository + UoWは過度な抽象化になりやすい。

**EF / EF Core以降、UoWが有効な場面（Part 3以降）**：DapperやADO.NETを使う環境。または、ボイラープレートと例外ベースのフロー制御を改善したい場合。

**現代の選択肢：**  

- **ベーシックUoW（パターンA）**：Dapper環境のシンプルで実績のある実装
- **スコープベースUoW（パターンB）**：ボイラープレート排除と型安全なフロー制御（Dapper環境向け独自実践パターン）
- **パイプラインUoW（パターンC）**：VSA + MediatR 環境での横断的トランザクション管理

最も重要な原則は変わりません。

> **「全部最新にすれば解決」ではなく、「プロジェクトの制約・チームのスキル・既存資産を踏まえた最善の選択をする」**

各パターンを理解しているからこそ、古いコードの意図が読め、混在環境で適切な設計判断ができ、新しいアーキテクチャを採用する際に「なぜそうなっているか」を説明できます。

### 20年以上の進化を経ても、UoWの核は変わらない

技術が変わり、アーキテクチャが変わり、UoWの実装形態は大きく変化しました。しかし次の3点は一度も変わっていません。

1. **トランザクション境界の管理**：いつ開始し、いつコミット/ロールバックするか
2. **原子性の保証**：すべて成功するか、すべて失敗するか
3. **データ整合性の維持**：複数の操作を1つの論理単位として扱う

変わったのは「**誰がこの責務を担うか**」です。

ADO.NET時代はUoWクラス自身が抱えていた。Entity Frameworkの登場でDbContextに移った。Dapper環境ではサービス層に戻った。そしてVSA + MediatRの時代に、Pipeline Behaviorへと移譲された。

責務の場所は変わり続けています。でも責務そのものは、2002年にFowlerが定義したときから変わっていません。

## 参考文献・出典

この記事の主張の根拠となる一次資料・公式ドキュメントをパート別に示します。

### Part 1：古典的 Unit of Work（ADO.NET時代）

**UoW パターンの原典定義**  

- Martin Fowler. *Unit of Work* — Patterns of Enterprise Application Architecture カタログ (2003)  
  <https://martinfowler.com/eaaCatalog/unitOfWork.html>
  本記事のPart 1で引用した変更追跡・Identity Map・トランザクション管理の3責務の出典。

**Repository パターンの原典定義**  

- Martin Fowler. *Repository* — Patterns of Enterprise Application Architecture カタログ (2003)  
  <https://martinfowler.com/eaaCatalog/repository.html>

**参考書籍**  

- Martin Fowler. *Patterns of Enterprise Application Architecture*. Addison-Wesley, 2002. ISBN: 978-0321127426
- Eric Evans. *Domain-Driven Design: Tackling Complexity in the Heart of Software*. Addison-Wesley, 2003. ISBN: 978-0321125217

### Part 2：Entity Framework 登場期

**「DbContextはUoWパターンを実装している」の明示的な記述**  

- Microsoft. *Design the infrastructure persistence layer* — .NET Microservices Architecture for Containerized .NET Applications  
  <https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design>
  原文：*"EF DbContext implements both the Repository and the Unit of Work patterns."*

- Microsoft. *Implementing the infrastructure persistence layer with Entity Framework Core*  
  <https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-implementation-entity-framework-core>
  原文：*"The Entity Framework DbContext class is based on the Unit of Work and Repository patterns and can be used directly from your code."*

**SaveChanges が暗黙的にトランザクションを張ることの公式記述**  

- Microsoft. *Working with Transactions - EF6*  
  <https://learn.microsoft.com/ef/ef6/saving/transactions>
  原文：*"whenever you execute SaveChanges() to insert, update or delete on the database the framework will wrap that operation in a transaction."*  
  ※ EF Core でも同じ動作が継承されている。

**EF時代の標準的な Generic Repository + UoW 実装**  

- Tom Dykstra (Microsoft). *Implementing the Repository and Unit of Work Patterns in an ASP.NET MVC Application* (2012)  
  <https://learn.microsoft.com/aspnet/mvc/overview/older-versions/getting-started-with-ef-5-using-mvc-4/implementing-the-repository-and-unit-of-work-patterns-in-an-asp-net-mvc-application>
  EF5時代のGeneric Repository + UoWの標準的実装例。Part 2の「当時の実装スタイル」の参考。

**Generic Repository に対する懐疑論（Jimmy Bogard のコメント）**  

- 上掲の Microsoft .NET Microservices eBook に引用  
  <https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design>
  原文：*"I'm really not a fan of repositories, mainly because they hide the important details of the underlying persistence mechanism. [...] Going CQRS meant that we didn't really have a need for repositories any more."*

### Part 3 / パターン A：ベーシック UoW（Dapper 環境）

**Dapper（軽量ORMとして普及した背景）**  

- StackExchange. *Dapper* — GitHub  
  <https://github.com/DapperLib/Dapper>

**ASP.NET Core DI コンテナ（パターンAの設計を可能にした技術）**  

- Microsoft. *Dependency injection in ASP.NET Core*  
  <https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection>

### パターン B：スコープベース UoW（筆者オリジナル設計）

> このパターンは既存の文献に直接の出典を持たない筆者の独自設計です。
> 以下はその設計思想の背景となった概念の出典です。

**Railway Oriented Programming（Result型によるエラーハンドリングの概念）**  

- Scott Wlaschin. *Railway Oriented Programming* — F# for fun and profit (2014)  
  <https://fsharpforfunandprofit.com/posts/recipe-part2/>
  Result型を用いた関数型エラーハンドリングのパターン。パターンBのフロー制御設計に影響を与えた概念。

**Result型ライブラリ**  

- FluentResults: <https://github.com/altmann/FluentResults>
- ErrorOr: <https://github.com/amantinband/error-or>
- LanguageExt: <https://github.com/louthy/language-ext>
- CSharpFunctionalExtensions: <https://github.com/vkhorikov/CSharpFunctionalExtensions>

### パターン C：パイプライン UoW（VSA + MediatR 環境）

**Vertical Slice Architecture の提唱**  

- Jimmy Bogard. *Vertical Slice Architecture* (2018)  
  <https://www.jimmybogard.com/vertical-slice-architecture/>
  パターンCの設計思想の直接的な一次ソース。レイヤードアーテクチャからの脱却とCQRSとの関係を論じている。

**MediatR（Pipeline Behaviorsの実装ライブラリ）**  

- Jimmy Bogard. *MediatR* — GitHub  
  <https://github.com/jbogard/MediatR>

**参考：軽量エンドポイントライブラリ**  

- Carter: <https://github.com/CarterCommunity/Carter>
- FastEndpoints: <https://fast-endpoints.com/>

### 参考：実装例・アーキテクチャガイド

- Jimmy Bogard. *ContosoUniversity (.NET Core Pages版)* — GitHub  
  <https://github.com/jbogard/ContosoUniversityDotNetCore-Pages>
- Microsoft. *eShopOnContainers - .NET Microservices Sample* — GitHub  
  <https://github.com/dotnet-architecture/eShopOnContainers>
- Microsoft. *.NET Application Architecture Guides*  
  <https://dotnet.microsoft.com/learn/dotnet/architecture-guides>
