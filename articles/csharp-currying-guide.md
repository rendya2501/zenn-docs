---
title: "C#におけるカリー化（Currying）"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp","関数型プログラミング"]
published: true
---

## 1. カリー化とは？

カリー化（Currying）とは、複数の引数を取る関数を、**1つの引数を取る関数を返す関数の連鎖**に変換するテクニックです。

通常の関数では、すべての引数を一度に受け取って処理しますが、カリー化された関数では、**1つの引数を受け取ると、新しい関数を返す**という形式を取ります。

この概念は、関数型プログラミングの分野でよく使われ、C#でも `Func<>` を活用することでカリー化を実現できます。

## 2. 基本的なカリー化の例

### 通常の関数

例えば、2つの整数を加算する関数は、通常以下のように書きます。

```csharp
int Add(int x, int y) {
    return x + y;
}
```

この関数は、`x` と `y` の両方の引数を受け取り、加算した結果を返します。

### カリー化した関数

この `Add` 関数をカリー化すると、以下のようになります。

```csharp
Func<int, Func<int, int>> CurriedAdd = x => y => x + y;
```

この `CurriedAdd` は、

1. まず `x` を受け取り、
2. `y` を受け取る関数を返し、
3. `x + y` を計算する

という形になっています。

カリー化された関数の呼び出し方法は以下のようになります。

```csharp
var add5 = CurriedAdd(5); // 5 を部分適用
int result = add5(3);      // 5 + 3 = 8
Console.WriteLine(result); // 8
```

または、一気に適用することもできます。

```csharp
int result2 = CurriedAdd(2)(4); // 2 + 4 = 6
Console.WriteLine(result2); // 6
```

## 3. カリー化を一般化する

カリー化を汎用的に行うために、**通常の関数をカリー化するメソッド** を作成できます。

```csharp
public static Func<T1, Func<T2, TResult>> Curry<T1, T2, TResult>(Func<T1, T2, TResult> func)
{
    return x => y => func(x, y);
}
```

このメソッドを使うと、通常の関数を簡単にカリー化できます。

```csharp
Func<int, int, int> add = (x, y) => x + y;
var curriedAdd = Curry(add);

var add10 = curriedAdd(10); // 10 を部分適用
Console.WriteLine(add10(5)); // 15
```

これにより、任意の2引数関数をカリー化することが可能になります。

## 4. 応用編：カリー化の実用例

カリー化は、**部分適用** を活用することで、再利用性の高い関数を作成できます。

### **例1: ログ出力関数**

```csharp
Func<string, Func<string, string>> Logger = level => message => $"[{level}] {message}";

var infoLogger = Logger("INFO");
var errorLogger = Logger("ERROR");

Console.WriteLine(infoLogger("システムが正常に起動しました"));
Console.WriteLine(errorLogger("重大なエラーが発生しました"));
```

出力：

``` txt
[INFO] システムが正常に起動しました
[ERROR] 重大なエラーが発生しました
```

このように、ログのレベルを固定した関数を作ることで、関数の再利用性が向上します。

### **例2: 数値変換関数のカリー化**

```csharp
Func<double, Func<double, double>> Multiply = x => y => x * y;

var doubleValue = Multiply(2); // 2倍する関数
var tripleValue = Multiply(3); // 3倍する関数

Console.WriteLine(doubleValue(10)); // 20
Console.WriteLine(tripleValue(10)); // 30
```

この方法を使えば、柔軟な変換関数を作ることができます。

## 5. カリー化のメリットと注意点

### メリット

1. **部分適用が可能** → 引数を一部固定した新しい関数を作成できる。
2. **関数の再利用性が向上** → 共通の処理を抽象化し、異なるコンテキストで使い回せる。
3. **コードの可読性が向上** → 処理を分割し、直感的な設計ができる。

### 注意点

- **C#はカリー化を標準サポートしていない** → `Func<>` を駆使する必要がある。
- **可読性の低下に注意** → 過度なカリー化は逆に理解しにくいコードを生むことがある。
- **パフォーマンスへの影響** → 関数オブジェクトを多く生成するため、メモリ使用量が増える可能性がある。

## 6. まとめ

- **カリー化とは？**
  - 関数を **1引数ずつ適用する形** に変換するテクニック。
  - `Func<>` を使うことでC#でも実現可能。

- **基本例と使い方**
  - `Func<int, Func<int, int>> CurriedAdd = x => y => x + y;`
  - `CurriedAdd(5)(3); // 8`

- **カリー化を汎用化するメソッド**
  - `Func<T1, Func<T2, TResult>> Curry<T1, T2, TResult>(Func<T1, T2, TResult> func)`

- **応用例**
  - ログ関数のカリー化 (`Logger("INFO")("Message")`)
  - 数値変換関数 (`Multiply(2)(10) // 20`)

- **メリットと注意点**
  - 部分適用の活用、再利用性向上
  - 過度なカリー化は可読性とパフォーマンスに影響

C#は関数型言語ではありませんが、**カリー化を活用することで柔軟な関数設計が可能** になります。特に **部分適用を活用した関数の再利用** に有効なので、場面に応じて取り入れてみましょう！
