---
title: '.NET と .NET Framework でエラーメッセージが違って悩んだ記録'
emoji: '🌊'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['csharp']
published: true
---

# 個人的な学び

- 動作環境を調査対象環境とあわせて調査すること

## 起きてたこと

```csharp
dynamic someString = "fetched data expected to be iterable object";
foreach(var k in someString["body"]) // <-- here
{
    Console.WriteLine(k);
}
```

コアの問題として `dynamic` 型の変数が予期せず `string` になっていたことがあり、過去事象からこの問題を疑った。

ただ、このとき上記の `here` のところで

```text
The best overloaded method match for 'string.this[int]' has some invalid arguments
```

というエラーが起きており、このメッセージが疑っていた問題に対して、すぐに噛み合って納得できなかった。

### 考えたこと

`variable["key"]` の `variable` が `string` であることを疑ってはいたものの(確信がなかったこともあり)どうにもエラーとあわないと一旦スルーしてしまった。
しかし `string.this[int]` は [string 型のインデクサ](https://source.dot.net/#System.Private.CoreLib/src/libraries/System.Private.CoreLib/src/System/String.cs,741)を指していることは当然として、実際のコードで `"string"[0]` のようなコードはなく(そもそも char を取る場面がなかった)、しばらくコードを彷徨うことになった。

他の可能性が考えにくかったので、あらためて上記のコードブロックを疑い、ローカルでは次のように確認したところエラーが違っていた。

```csharp
var someString = "fetched data expected to be iterable object";
foreach(var k in someString["body"])
{
    Console.WriteLine(k);
}
```

```text
Argument 1: cannot convert from 'string' to 'int'
```

そこで、フレームワーク違いを疑って、型とフレームワークの組み合わせで確認したところエラーはそれぞれ次のようになることがわかった。

![](/images/fb7fc0adf8cb5b.png)

エラーメッセージ的には、`cannot convert from 'string' to 'int'` と言ってくれたほうが個人的にはわかりやすいと感じた。
(謎が解けて振り返ると、まあ分からないではないメッセージではあるが、過去の事象から疑いが見えてないともっと悩んだだろうなと思う)

ただ dynamic では ` CallSite`` を経由して動的にメソッドを選択する都合上  `some invalid` としか言えないことは想像に難くない。

## まとめ

調査がうまく進まなかった原因は 2 点

1. 実環境では、変数は関数の呼び出しの結果を代入していたので `var` は `dynamic` に推論されていた(上記検証コードでは `var` は`string` に推論される)
2. ローカルでは .NET 7 を利用していた(問題が発生していた環境は .NET Framework 4.7 系)

今回は [.NET Fiddle](https://dotnetfiddle.net/)を利用することで手軽に差異を確認できたので、環境をあわせての確認に注意したい。
