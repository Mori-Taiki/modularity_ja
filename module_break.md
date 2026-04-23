こんにちは、エンジニアのT.Mです。今日は私の代わりに、私の中のファジーなペルソナ🫥が記事を書いてくれるそうです。


ファジー🫥：「強固なアーキテクチャって、堅苦しいよね？」「なんだか冗長に見えるし、何の意味があって分かれているのか、理解しがたいことばっかり！」「業務では絶対やれないこと、ブログではやっちゃおう！！！」

なんだか、気分が悪くなってきました。しかし、ここは温かい目で見守ることにします。

🫥「ここからは天才プログラマーであるところの俺様が、シンプルなコードってのを見せてやるよ」

# 堅苦しく、長いコード
🫥今回取り組むのはこのコード。どうやらチップ？とかいうのを計算するためのコードらしい。
```cs
namespace Billing.Tipping;

public interface ITipCalculator
{
    Money CalculateTip(Subtotal subtotal, TipRate rate);
}

public readonly record struct TipRate
{
    public decimal Value { get; }
    public TipRate(decimal value)
    {
        if (value < 0m || value > 1m)
            throw new ArgumentOutOfRangeException(nameof(value));
        Value = value;
    }
}

public sealed record Currency(string Code);
public sealed record Money(decimal Amount, Currency Currency);
public sealed record Subtotal(Money Value);

public sealed class PercentageTipCalculator : ITipCalculator
{
    public Money CalculateTip(Subtotal subtotal, TipRate rate) =>
        subtotal.Value with { Amount = subtotal.Value.Amount * rate.Value };
}
```

🫥なんかクラスっぽいのがたくさんあるし、よく見るとインターフェース？とかレコード？とか色々あってややこしい。
どうやら、チップというのは、購入金額に一定の割合を乗算すればいいらしいが、**それをやるだけにこんなたくさんのコードが必要なのか？**
もっと"シンプル"にしてやろう。
なんか後ろから、「お前は"Simple"と"Easy"を混同している」という声が聞こえるが、無視して進めることにする。

https://zenn.dev/loglass/articles/6b56dac587f02e

## インターフェース？それ要るの？
🫥とにもかくにも、インターフェースとかいう奴がよくわからないので消しちまおう。
```cs
namespace Billing.Tipping;

public sealed class TipCalculator
{
    public Money CalculateTip(Subtotal subtotal, TipRate rate) =>
        subtotal.Value with { Amount = subtotal.Value.Amount * rate.Value };
}
```
🫥これでヨシ！結局やってること同じだよね？
え？「チップが常に%計算の米国式とは限らないだろ、日本ではサービス料込みとして計算するんだ」「ポリシーの差し替えが見込まれるからこうしてたのに」って？
ちょっと何言ってるかよくわからないな…まだそんなコードなかったよ？

「契約結合がモデル結合に変化」した…？何のことをおっしゃっているのだか？

## なんでわざわざ値をラップするわけ？
🫥レコード？とかいうのは、クラスの亜種で、なんか値を使うときに使うヤツらしい？でも、そもそも値ならプリミティブ型でよくない？
```cs
namespace Billing.Tipping;

public static class TipCalculator
{
    public static decimal CalculateTip(decimal subtotal, decimal rate)
        => subtotal * rate;
}
```
🫥ほら、これでMoneyとかTipRate とかいちいち宣言しなくて良くなりましたよ？俺って天才…？
え？「レートが0~1の範囲内かどうかのチェックはどこへ行ったんだ」って？そんなの…呼び出すときにチェックしなよ…

分かった分かった、じゃあメソッド内でチェックすればいいんでしょ？
```cs
namespace Billing.Tipping;

public static class TipCalculator
{
    public static decimal CalculateTip(decimal subtotal, decimal rate)
    {
        if (rate < 0m || rate > 1m)
            throw new ArgumentOutOfRangeException(nameof(rate));
        return subtotal * rate;
    }
}
```
🫥ほら、これで満足？
何？「チップのレートが正しいかどうかを計算側でチェックするのはおかしい」って…？何が違うのかよくわからないな…
え…？「他にもレートを使う場合にそっちにもガード節を書く必要があるだろ」…？どういうこと…？
```cs
public static class TipCalculator
{
    public static decimal CalculateTip(decimal subtotal, decimal rate)
    {
        if (rate < 0m || rate > 1m)
            throw new ArgumentOutOfRangeException(nameof(rate));
        return subtotal * rate;
    }

    // 追加: 良いサービスには 2% のチップを上乗せする
    public static decimal CalculateTipForExcellentService(decimal subtotal, decimal rate)
    {
        return subtotal * (rate + 0.02m);   // ← ガード節を書き忘れ
    }
}
```

```cs
// 8% のつもりで 8 を渡してしまった
TipCalculator.CalculateTip(1000m, 8m);
// → ArgumentOutOfRangeException で弾かれる

TipCalculator.CalculateTipForExcellentService(1000m, 8m);
// → 1000 * (8 + 0.02) = 8020円 のチップ（静かに通る）
```
🫥いやいや、そんなこと今考えてもしょうがないでしょ？だって今そんな関数は無いんだし。

何…？まだあるの？「チップのレートと税金のレートを取り違えたらどうするんだ」って…？まだそんな要件ないじゃん、それに変数名ちゃんとしてればそんなこと起こらないよ、人間を信じなって。

「モデル結合が機能結合に変化」した…？よくわかんないけど、難しい言葉でごまかそうとするの止めてもらえる？

## もっと"汎用的"にしよう
🫥面白くないことに、その後チップだけじゃなくて、税金も計算しろって言われた。（新たな要件）
でもチップの計算とほとんど同じ処理なんだよなぁ...
そうだ、Tipなんて名前があるから誤用だと思われるんだ、もっと"汎用的"にしてやろう。

```cs
namespace Utils;

public static class Calculator
{
    public static decimal Calculate(decimal x, decimal rate)
    {
        if (rate < 0m || rate > 1m)
            throw new ArgumentOutOfRangeException(nameof(rate));
        return x * rate;
    }
}
```
``` cs
var tip = Calculator.Calculate(subtotal, 0.15m);   // チップ
var tax = Calculator.Calculate(subtotal, 0.08m);   // 消費税
```
🫥ほら、これってDRY（Don't Repeat Yourself）原則に則っているよね？
え…？「DRY原則は抽象化の事であって、共通化の事ではない？」何が違うのよ？

「別の意味の処理が密結合するだろ」…？
```cs
// 1. マイナス税の計算をしたいのでレンジを広げた
namespace Utils;

public static class Calculator
{
    public static decimal Calculate(decimal x, decimal rate)
    {
        // 還付にも対応するため下限を撤廃
        if (rate > 1m)
            throw new ArgumentOutOfRangeException(nameof(rate));
        return x * rate;
    }
}
```
```cs
// 2. マイナスのチップが通るようになってしまった
// 税チームの想定: 還付率を渡す
Calculator.Calculate(1000m, -0.05m);   // -50円（還付として正しい）

// チップ側でうっかり同じ関数を呼ぶと…
Calculator.Calculate(subtotal, tipRate);   // tipRate = -0.05m でも素通り
// → 顧客に「マイナスのチップ」が発生し、支払額が勝手に減る
```
🫥いやいや、悲観的すぎるって、心配ごとの9割は実際には起きないらしいよ？
「もはや侵入結合だ」…？なんでそんな悲しそうな顔するの？

---

ハッ…！私は一体何を、なんだかすごく頭が痛い…

# 何が起こっていたのか
茶番は差し置いて、ファジー🫥なペルソナの感覚はこうです
- モジュールの契約を柔らかくしたい
- 早すぎる実装をしたくない
- とにかく全体を短くしたい

これは一概に否定すべき感覚ではないと思っています。
実際、インターフェースが過剰になることはありますし、すべての値に対して型(値オブジェクト)を定義すると、非生産的になるフェーズもあると思います。
流石に、侵入結合については、別々に書いた方がいいことが99%だとは思いますが。

大事なのは、どのようなリスクに備えた設計なのか知ることだと思います。
そして、それを一言で表現する用語を知っていれば、自分の中での認知負荷が低くなり、チームで共有されれば指摘もスムーズです。

すでにお気づきの方もいらっしゃると思いますが、本ブログのモジュール結合についての定義は、Vlad Khononovの『ソフトウェア設計の結合バランス』に基づいています。
https://amzn.asia/d/02wEkr8p

彼の定義ではモジュール結合は4つの段階に分類されます。(実際にはコナーセンス等を使ってそれぞれの段階の内部でもレベルがあるのですが、簡単のために今回はそこには踏み込みませんでした。)

モジュール間の結合は、より弱い順から以下の通りです。

* コントラクト結合
* モデル結合
* 機能結合
* 侵入結合

この記事では、あえてより疎である結合から、知識を漏出させていく形を取ってみました。
そのため、設計にこだわりのある方には読むのが苦痛の記事だったかもしれません。
この記事を下から読み直して、リファクタリングされていく気持ちよさを味わい、精神の安寧を得ていただければ幸いです。

| 下から読むと | 何が起きているか | 結合の移動 |
| --- | --- | --- |
| Utils → Billing.Tipping に戻す | ドメインの境界を引き直す | 侵入 → 機能 |
| TipRate を復活 | 不変条件を型に押し込める | 機能 → モデル |
| ITipCalculator を復活 | ポリシー差し替えを契約で吸収 |  モデル → 契約 |

ちなみに、Vlad Khononovのモジュール結合バランスに関する考えは、以下のリポジトリでAIエージェント用のskillsにもなっています。こちらを一読するだけでも大変勉強になるかと思います。本と合わせて非常におすすめです。
https://github.com/vladikk/modularity