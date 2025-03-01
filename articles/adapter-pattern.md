---
title: "【デザインパターン】Adapterパターン"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["designpattern","csharp"]
published: true
---

## はじめに

完全に個人用の勉強メモです。  
Adapterパターンについては調べればたくさん出てきますので、ここでは個人的に有用だと思われる実用的なサンプルをまとめて行きます。  
未完成なので都度、加筆修正していきます。  
後、物凄く雑です。  

<!-- --- -->

## パターン1

ものすごくバージョンの古いActiveReportがあったとします。  
※ActiveReportはGrapeCityが提供する.Net系の帳票ライブラリです。  

A,B,Cの帳票があって、どの帳票も同じプロパティとメソッドを持ちます。  
違うのはデザインだけです。  
呼び出すメソッドやプロパティが同じなので、インターフェースを嚙ませればいいじゃないと思うかもしれませんが、そのままインターフェースを噛ませるとデザイナーの関係もあって死にます。  
※古いバージョンはコードとデザイナーと他何かの3つのファイルが抱き合わさって動きます。  

``` cs : 元のコード
partial class NotAdapter
{
    void Do(string pattern, string hoge, string fuga)
    {
        if (pattern == "A")
        {
            // 同じプロパティとメソッドを持つのでインターフェースで解決しそうではある。
            // それで済むならアダプターパターンはいらない。
            var a = new A_ActiveReport();
            a.Hoge = hoge;
            a.Fuga = fuga;
            a.Run();
        }
        else if (pattern == "B")
        {
            var b = new B_ActiveReport();
            b.Hoge = hoge;
            b.Fuga = fuga;
            b.Run();
        }
        else
        {
            var c = new C_ActiveReport();
            c.Hoge = hoge;
            c.Fuga = fuga;
            c.Run();
        }
    }
}

// ActiveReportの仮の定義。  
// 実際の定義は隠蔽されている。  
public class A_ActiveReport
{
    public string Hoge { get; set; }
    public string Fuga { get; set; }
    public A_ActiveReport() { }
    public void Run() { }
}

public class B_ActiveReport
{
    public string Hoge { get; set; }
    public string Fuga { get; set; }
    public B_ActiveReport() { }
    public void Run() { }
}

public class C_ActiveReport
{
    public string Hoge { get; set; }
    public string Fuga { get; set; }
    public C_ActiveReport() { }
    public void Run() { }
}
```

そこでアダプターとしてワンクッションかませればいいじゃないという発想になります。  
今回は継承パターンで実装してみました。  

``` cs : 修正コード
// アダプターインターフェース
public interface IActiveReportAdapter
{
    public string Hoge { get; set; }
    public string Fuga { get; set; }
    public void Run() { }
}

// アダプターオブジェクト
public class A_ActiveReportAdapter : A_ActiveReport, IActiveReportAdapter { }
public class B_ActiveReportAdapter : B_ActiveReport, IActiveReportAdapter { }
public class C_ActiveReportAdapter : C_ActiveReport, IActiveReportAdapter { }

public class Adapter
{
    public void Do(string pattern, string hoge, string fuga)
    {
        // .net6で実装したのでswitch式使えるならこれで済む。  
        IActiveReportAdapter adapter = pattern switch
        {
            "A" => new A_ActiveReportAdapter(),
            "B" => new B_ActiveReportAdapter(),
            _ => new C_ActiveReportAdapter()
        };

        adapter.Hoge = hoge;
        adapter.Fuga = fuga;
        adapter.Run();
    }

    // .NetFramework 3の時代なのでswitch式もLinqもない場合は関数として切り出した方がよいだろう
    // SingletonでFactoryクラスとして定義するのもいいかも。
    public IActiveReportAdapter CreateActiveReportAdapter(string pattern)
    {
        switch (pattern)
        {
            case "A":
                return new C_ActiveReportAdapter();
            case "B":
                return new B_ActiveReportAdapter();
            default:
                return new C_ActiveReportAdapter();
        }
    }
}
```

<!-- --- -->

## 参考

[デザインパターン入門Adapterパターンについて](https://zenn.dev/komorimisaki422/articles/6ea4594bb875d2)  
[](https://ie.u-ryukyu.ac.jp/~e085739/java.it.6.html)
[](https://qiita.com/yutoo89/items/5bcdeb0e971477460366)
