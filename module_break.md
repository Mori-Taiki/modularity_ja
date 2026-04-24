こんにちは、エンジニアのT.Mです。今日は私の代わりに、私の中のYAGNI🫥なペルソナが記事を書いてくれるそうです。


YAGNI🫥：「強固なアーキテクチャって、堅苦しいよね？」「なんだか冗長に見えるし、何の意味があって分かれているのか、理解しがたいことばっかり！」「業務では絶対やれないこと、ブログではやっちゃおう！！！」

なんだか、気分が悪くなってきました。しかし、ここは温かい目で見守ることにします。

🫥「ここからは天才プログラマーであるところの俺様が、シンプルなコードってのを見せてやるよ」

> T.M. 流石に心配なので、後ろから口を出すことにします。

# 堅苦しく、長いコード

🫥今回取り組むのはこのコード。飲食店の会計を行うCheckoutモジュールで、チップ込みの合計金額を計算するためのコードらしい。

```cs
namespace Checkout;

// チップ計算 契約
internal interface ITipCalculator
{
    Money Calculate(Money subtotal, TipRate rate);
}

// チップ計算器
internal sealed class PercentageTipCalculator : ITipCalculator
{
    public Money Calculate(Money subtotal, TipRate rate) =>
        subtotal with { Amount = subtotal.Amount * rate.Value };
}

// 会計ユースケースの契約（ファサード）
public interface ICheckoutWorkflow
{
    Receipt Finalize(Order order);
}

// 会計ユースケースの実装
public sealed class CheckoutWorkflow(ITipCalculator tip) : ICheckoutWorkflow
{
    public Receipt Finalize(Order order)
    {
        var tipAmount = tip.Calculate(order.Subtotal.Value, order.TipRate);
        var total     = order.Subtotal.Value with { Amount = order.Subtotal.Value.Amount + tipAmount.Amount };
        return new Receipt(order.Id, total);
    }
}

// 値オブジェクトの定義　・・・省略・・・
```

🫥なんか長かったので一旦後半は省略、後ほどいい感じにしてやる。
そんで、このモジュールは以下のように使われている。

```cs
// 利用側：ICheckoutWorkflow のみ知っている
public class OrderController(ICheckoutWorkflow checkout)
{
    public Receipt Submit(Order order) => checkout.Finalize(order);
}
```

🫥なんか型がめちゃくちゃたくさんあるし、インターフェースも2つある。
どうやら、やっていることは「購入金額 × チップ率」を足して合計を返すだけらしいが、**それをやるだけにこんなたくさんのコードが必要なのか？**
もっと"シンプル"にしてやろう。

🫥 なんか後ろから、
> 「お前は"Simple"と"Easy"を混同している」

という声が聞こえるが、無視して進めることにしよう

（参考: [「Simple」と「Easy」はちがう。 - ログラスEngineers Blog](https://zenn.dev/loglass/articles/6b56dac587f02e)）

## このインターフェース、要る？

🫥 `CheckoutWorkflow` って結局のところ、中で `PercentageTipCalculator` を呼んでいるだけだ。利用側はインターフェース越しに一段噛まされて、実際の計算器に辿り着くまでに迷子になる。**一段減らせばシンプルだろ？**

🫥 何？
> 「これはファサード（窓口）パターンと言って、モジュールの玄関口を一つにして複雑さを減らすんだ」

って？よくわからないけど`F2`で実装に飛べないのはちょっと…　

🫥 え？

> 「選択して`Ctrl + F12`で飛べる」

って？そんなのいちいち覚えてらんないよ…俺は"慣れた"手段で仕事がしたいんだ。

```cs
// Checkout モジュール：ファサード（ICheckoutWorkflow）を消す
namespace Checkout;

// public に格上げ：内部の「部品」が外から見えるようになる
public interface ITipCalculator
{
    // 責務を少し上げて、チップ込みの合計を返すようにしておく
    Money ApplyTo(Money subtotal, TipRate rate);
}

public sealed class PercentageTipCalculator : ITipCalculator
{
    public Money ApplyTo(Money subtotal, TipRate rate)
    {
        var tipAmount = subtotal.Amount * rate.Value;
        return subtotal with { Amount = subtotal.Amount + tipAmount };
    }
}
```

```cs
// 利用側：部品を直接呼ぶ
public class OrderController(ITipCalculator tip)
{
    public Receipt Submit(Order order)
    {
        var total = tip.ApplyTo(order.Subtotal.Value, order.TipRate);
        return new Receipt(order.Id, total);
    }
}
```

🫥これでヨシ！結局やってること同じだよね？1段減った分、スッキリしただろ？

🫥え？なんか文句あるの？

> 将来 `ITaxCalculator` が増えたら、利用側はそれも自分で呼び足さなきゃいけないだろ

> 計算器を `IChargeCalculator` みたいにまとめて再設計したら、利用側が壊れる

🫥 そうだとしても、また**必要になったら**、そのときにファサードとやらを足せばいいだろ？今そんな要件ないんだし。

> ファサードがあった頃は、Checkout チームが内部の**部品構成**を好きに動かしても利用側を壊さなかった

> インターフェースをはがしたことで、**どの計算器がどう並んでいるか**という内部構造が利用側に漏出した

🫥何言ってるかよくわからないんだけど…？

> 「**契約結合**が**モデル結合**に変化した」

🫥…？何のことをおっしゃっているのだか？

## なんでわざわざ値をラップするわけ？

🫥お次は、先ほど省略したここのコードだ。謎に値をラップしている。

```cs
namespace Checkout;

// 値オブジェクト
public sealed record Money(decimal Amount, string Currency);
public sealed record Subtotal(Money Value);
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
public sealed record Order(OrderId Id, Subtotal Subtotal, TipRate TipRate);
public sealed record Receipt(OrderId OrderId, Money Total);
```

🫥レコード？とかいうのは、クラスの亜種で、なんか値を使うときに使うヤツらしい？でも、そもそも**値**ならプリミティブ型でよくない？

🫥インターフェース経由で呼ぶのも面倒だし、ついでに static メソッドにしちまおう。

```cs
namespace Checkout;

public static class TipCalculator
{
    public static decimal Calculate(decimal subtotal, decimal rate)
    {
        if (rate < 0m || rate > 1m)
            throw new ArgumentOutOfRangeException(nameof(rate));
        return Math.Round(subtotal * rate, 2);
    }
}
```

🫥ほら、これで `Money` とか `TipRate` とかいちいち宣言しなくて良くなりましたよ？俺ってやっぱ天才…？

🫥え？

>「レートが0〜1の範囲内かどうか のチェックはレートという値自身が保証すべき」

って？どこでチェックしても同じでしょ。

🫥何？まだ文句あるの？

> 「それに、`TipRate` と `TaxRate` が型で区別できなくなった」「取り違えたときにコンパイラで気づけないだろ」

…？そんなの変数名ちゃんとしてればそんなこと起こらないよ、人間を信じなって。

```cs
// 利用側：decimal の意味と 0〜1 の制約を"暗黙的に"知っている必要がある
public class OrderController
{
    public Receipt Submit(Order order)
    {
        var tipRate = 0.15m;
        var taxRate = 0.08m;
        var tip     = TipCalculator.Calculate(order.SubtotalAmount, taxRate);
        // ↑ うっかり taxRate を渡しても型的にはまったく同じ。コンパイラで気づけない
        var total   = Math.Round(order.SubtotalAmount + tip, 2);  // 丸め方針も利用側で実装が必要になる
        return new Receipt(order.Id, new Money(total, "JPY"));
    }
}
```

🫥別にそんなの一回気をつければいいんだし、どうってことないでしょ？

> 「型に閉じ込められていた**『レートは0〜1』という不変条件の知識**が、利用側に漏れ出した」

> 「今や利用側は、Checkout内部のビジネスルール（制約・丸め方針）を**自分で覚えている必要がある**」

> 「**モデル結合**が**機能結合**に変化した」

…？よくわかんないけど、難しい言葉でごまかそうとするの止めてもらえる？

## コピペでいいよコピペで、既存機能触りたくないし

…しばらく経って、返金（Refund）機能を作ることになった。返金でも小計・チップ・丸めの扱いが必要だ。

🫥既存の機能触るとテストする個所が増えるなぁ… Checkout のロジックに似てるし、コピペでよくない？

```cs
// 返金サービス：同じロジックを独自にコピー
namespace Refunds;

public class RefundService
{
    public Refund Calculate(Order order)
    {
        // Checkout 側のルールをコピペ。ただし返金なので 0〜1 のガードは"いらない気がする"ので外した
        var tip   = Math.Round(order.SubtotalAmount * order.TipRateValue, 2);
        var total = Math.Round(order.SubtotalAmount + tip, 2);
        return new Refund(order.Id, total);
    }
}
```

🫥コピペだから動作は同じ。完璧！え…？

> "割引→税→チップの順で適用する"や"小数第2位で丸める"みたいな**ビジネスルールが複数の場所に重複**している

>仕様変更が発生したときにすべてのコピーに反映できる保証はないだろ

🫥…？そんなことそうそうないって。

> 「"重複した機能"は**機能結合**の中でも最も深刻」

🫥…？ちゃんと調べればその時に気づくでしょ。

## じゃあ共通化すればいいんでしょ？
🫥分かった分かった、そんなに言うなら同じ処理を纏めておけばいいんでしょ？

```cs
// Util.cs：Checkout と Refund の"似たような計算"を1つに纏めた
namespace Shared;

public static class PaymentUtil
{
    // rate のガードが要るかどうかは呼び出し側の事情なのでフラグにした
    public static decimal Calculate(
        decimal amount,
        decimal rate,
        bool requireRateGuard = true)
    {
        if (requireRateGuard && (rate < 0m || rate > 1m))
            throw new ArgumentOutOfRangeException(nameof(rate));

        var applied = Math.Round(amount * rate, 2);
        return Math.Round(amount + applied, 2);
    }
}
```

```cs
// Checkout 利用側：デフォルト引数を暗黙的に利用する
public class OrderController
{
    public Receipt Submit(Order order)
    {
        var total = PaymentUtil.Calculate(order.SubtotalAmount, order.TipRateValue);
        return new Receipt(order.Id, new Money(total, "JPY"));
    }
}

// Refund 利用側：「うちは要件が違うので」とフラグで切り替え
public class RefundService
{
    public Refund Calculate(Order order)
    {
        var total = PaymentUtil.Calculate(
            order.SubtotalAmount,
            order.TipRateValue,
            requireRateGuard: false);  // 返金ではガード不要（と思って外した）
        return new Refund(order.Id, total);
    }
}
```

🫥ほら、これで満足？これってDRY(Don't Repeat Yourself)原則っていうんだろ？知ってるよ？

🫥え？
> 「違うものを同じところにまとめるくらいなら別々の方がマシだった、会計と返金のどっちかがUtilを変更したらもう片方が引きずられてバグる」

> 「しかも `bool requireRateGuard` で**利用側に内部の分岐を決めさせている**。これは**制御結合（control coupling）**と呼ばれる、機能結合の中でも特に厄介な形だ」

もう、まとめりゃいいのか別々にすりゃいいのかはっきりしてくれよ…

🫥何？まだあるの？
>「共通化と抽象化は違う、DRY原則は抽象化の話だ」…？「じゃあ何で正しく抽象化されていた値オブジェクトを消したんだよ」

…？その抽象化とやらの話を延々するより、さっさと作るほうが効果的だろ？

## APIできるのなんか待ってらんない、DB直で覗いちゃおう

🫥人手が足りないってので別のチームに移動した。…今度は「現在保留中の会計一覧」を表示するダッシュボードが必要らしい。

🫥 Checkoutチームに「保留中会計を返すAPIくれ」ってお願いしたら、「仕様整理にちょっと時間ください」って返された…こっちだって急いでるのに、前のよしみで融通してくれてもいいじゃん。

🫥 たしか、`PendingReceipts`っていうテーブルだったよな…

```cs
// Checkoutモジュール：内部のテーブル
namespace Checkout;

internal class CheckoutDbContext : DbContext
{
    public DbSet<PendingReceiptRow> PendingReceipts { get; set; }
}

internal class PendingReceiptRow
{
    public Guid OrderId { get; set; }
    public decimal Subtotal { get; set; }
    public decimal Tax { get; set; }
    public decimal Tip { get; set; }
}
```

🫥 なんかすげぇ急かされてるし、もう、直接SELECTすれば早くない？

```cs
// ダッシュボード側：Checkoutの内部テーブルを生SQLで直接読む
namespace Dashboard;

public class PendingCheckoutsView(IDbConnection db)
{
    public IReadOnlyList<PendingSummary> GetAll()
    {
        // Checkoutチームが"内部"と思っているテーブルを横から覗く
        var rows = db.Query<(Guid OrderId, decimal Subtotal, decimal Tax, decimal Tip)>(
            "SELECT OrderId, Subtotal, Tax, Tip FROM checkout.PendingReceipts");

        return rows
            .Select(r => new PendingSummary(r.OrderId, r.Subtotal + r.Tax + r.Tip))
            .ToList();
    }
}
```

🫥 SQL書けば一瞬じゃん。API待ってらんないよ。

🫥 え？
> 「Checkoutチームは、`PendingReceiptRow` が外から参照されていることを**知らない**」

> 「カラム名を変えたり、テーブルを別スキーマに移しただけで、ダッシュボードが**サイレントに**壊れる。コンパイルは通るから、本番で『お客様の保留会計が突然表示されない』という形で事故になる」

🫥 …じゃあCheckoutチームに『テーブル変えないで』って言っとけばいいじゃん。

> 「それはもう、Checkoutチームが自分のモジュールを直すたびに**"他に誰が俺の内部を見てるか"を毎回調査しなきゃいけない**ってことだ」

> 「**機能結合**から**侵入結合**に変化した。これは最も頑固な結合だ」

🫥 ……

---

ハッ…！私は一体何を、なんだかすごく頭が痛い…

# 何が起こっていたのか

茶番はさておき、YAGNI🫥の感覚はこうです。

- モジュールの契約を柔らかくしたい
- 早すぎる実装をしたくない
- とにかく全体を短くしたい

これは一概に否定すべき感覚ではないと考えます。
実際、🫥の主張には **特定の条件下では正しいもの** がたくさんあります。問題は、彼が一貫して「このコードは変わらない」「この要件は増えない」という ***低変動性を暗黙の前提*** としていることです。

## 結合強度の4段階

すでにお気づきの方もいらっしゃると思いますが、本ブログのモジュール結合についての定義は、Vlad Khononov の『ソフトウェア設計の結合バランス』に基づいています。

[ソフトウェア設計の結合バランス（Vlad Khononov）— Amazon](https://amzn.asia/d/02wEkr8p)

彼の定義では、モジュール結合は以下の4つの段階に分類されます。(実際にはコナーセンス等を使ってそれぞれの段階の内部でもレベルがあるのですが、簡単のために今回はそこには踏み込みませんでした。)

YAGNI🫥の一手ずつ、コードは「より多くの知識」を利用側に漏らしていきました。Vlad Khononov の Balanced Coupling モデルでは、モジュール間で共有される知識の量によって、結合を4段階に分類します（弱い順）:

| 段階 | 利用側が Checkout について知っていること | 典型シグナル | 変更の波及 |
| --- | --- | --- | --- |
| **契約結合 (Contract)** | `Finalize(Order) → Receipt` のシグネチャだけ | Facade / DTO / Open-host service | 契約面を守れば内部は自由 |
| **モデル結合 (Model)** | 内部の計算者インターフェース群 | `internal` の interface が `public` に漏れる | 部品構成の変更が利用側に波及 |
| **機能結合 (Functional)** | 計算順序・丸め・不変条件などの**ビジネスルール** | 同じロジックが複数箇所にコピー | ルール変更が全コピーに追随しないと不整合 |
| **侵入結合 (Intrusive)** | 内部テーブル構造・`internal` メソッドの存在 | `InternalsVisibleTo`, 他モジュールの DB 直読 | 非公開の変更が**サイレントに**利用側を壊す |

Vlad Khononov のモジュール結合バランスモデルでは、結合を評価するもう一つの軸として **変動性（Volatility）** があります。仮に結合が強くても、そのコンポーネントが「事実上一生変わらない」なら実害はほぼ発生しません。逆に結合が弱くても、頻繁に変わる領域ではほんの少しの知識漏出が継続的なツラみになります。つまり、**YAGNIは変動性が低い領域では正解、高い領域では負債になりうる**ということだと理解しています。

Checkoutのようなビジネスの中核（コアサブドメイン）は、まさに変動性が高い領域の典型です。🫥が冒頭で"堅苦しい"と呼んだ構造の多くは、変動性に備えるための**投資**でした。契約・値オブジェクト・Facade──いずれも「将来の変更を、利用側に漏らさずに内側で吸収する」ためのしくみです。

# 下から読むと、リファクタリング

この記事では、あえてより弱い結合から、知識を漏出させて強固な結合に変化されていく形を取ってみました。
そのため、設計にこだわりのある方には、読むのが苦痛の記事だったかもしれません。
この記事を下から読み直して、リファクタリングされていく気持ちよさを味わい、精神の安寧を得ていただければ幸いです。

| 下から読むと | 何が起きているか | 結合の移動 |
| --- | --- | --- |
| 共有DBの直読をやめ、Checkoutに公開APIを要求する | 内部テーブルへの侵入を、契約の向こう側へ戻す | 侵入結合 → 機能結合(制御結合) |
| `PaymentUtil` の偶発的な共通化を解体し、Refund の重複ロジックも取り除く | 「似ているだけで違うもの」を同居させていた暗黙の共有をほどく | 機能結合(制御結合) → 機能結合(重複した機能) |
| `TipRate` / `Money` / 丸め方針を型と責務に戻す | 不変条件とビジネスルールを型・1箇所に押し込める | 機能結合(重複した機能) → モデル結合 |
| `ICheckoutWorkflow` Facade を復活し、計算器を `internal` に戻す | 内部構造の露出を契約で包み直す | モデル結合 → 契約結合 |

## ※補足

今回考慮しなかったのですが、実際にはもう一つ、"**距離**"という重要な概念があり、これを紹介することなしには、本当に適切なコンポーネント結合を定義することはできません。例えば、

- コンポーネント同士が同じパッケージにあるのか、ライブラリになっているのか
- 同期処理なのか非同期処理なのか
- ライフサイクルは同じなのか、チームは別々なのか同じなのか

こういった要素も含めて初めて適切なモジュール結合の「**バランス**」を見極めることができます。しかし紙面の関係上、ここでは紹介できませんので、関心のある方はぜひ実際に本をお読みいただければ幸いです。多くの有名なエンジニアのお墨付きである、素晴らしい本です。

## おまけ

ちなみに、Vlad Khononov のモジュール結合バランスに関する考えは、以下のリポジトリで AI エージェント用の skills にもなっています。こちらを一読するだけでも大変勉強になるかと思います。本と合わせて非常におすすめです。

[vladikk/modularity — GitHub](https://github.com/vladikk/modularity)
