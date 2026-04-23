こんにちは、エンジニアのT.Mです。今日は私の代わりに、私の中のYAGNI🫥なペルソナが記事を書いてくれるそうです。


YAGNI🫥：「強固なアーキテクチャって、堅苦しいよね？」「なんだか冗長に見えるし、何の意味があって分かれているのか、理解しがたいことばっかり！」「業務では絶対やれないこと、ブログではやっちゃおう！！！」

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

// 米国式: 購入金額に一定の割合を乗算する
public sealed class PercentageTipCalculator : ITipCalculator
{
    public Money CalculateTip(Subtotal subtotal, TipRate rate) =>
        subtotal.Value with { Amount = subtotal.Value.Amount * rate.Value };
}

// 日本式: サービス料として一定率を加算する
public sealed class JapaneseServiceChargeCalculator : ITipCalculator
{
    private static readonly decimal ServiceChargeRate = 0.10m;
    public Money CalculateTip(Subtotal subtotal, TipRate _) =>
        subtotal.Value with { Amount = subtotal.Value.Amount * ServiceChargeRate };
}
```

そして、このモジュールは以下のように使われます。
```cs
// 利用側: ITipCalculator だけを知っていればいい
public class CheckoutService(ITipCalculator tipCalculator)
{
    public Receipt Finalize(Order order)
    {
        var tip = tipCalculator.CalculateTip(order.Subtotal, order.TipRate);
        return new Receipt(order, tip);
    }
}

// 店舗のロケールに応じて DI で差し替え
services.AddScoped<ITipCalculator, PercentageTipCalculator>();          // 米国店舗
services.AddScoped<ITipCalculator, JapaneseServiceChargeCalculator>();  // 日本店舗
```

🫥なんかクラスっぽいのがたくさんあるし、よく見るとインターフェース？とかレコード？とか色々あってややこしい。
どうやら、チップというのは、購入金額に一定の割合を乗算すればいいらしいが、**それをやるだけにこんなたくさんのコードが必要なのか？**
もっと"シンプル"にしてやろう。
なんか後ろから、「お前は"Simple"と"Easy"を混同している」という声が聞こえるが、無視して進めることにする。

https://zenn.dev/loglass/articles/6b56dac587f02e

## インターフェース？それ要るの？
🫥とにもかくにも、インターフェースとかいう奴がよくわからないので消しちまおう。
ついでに日本式も要らないから米国式だけ残す。日本にチップの文化はないからね。
```cs
namespace Billing.Tipping;

public sealed class TipCalculator
{
    public Money CalculateTip(Subtotal subtotal, TipRate rate) =>
        subtotal.Value with { Amount = subtotal.Value.Amount * rate.Value };
}
```
🫥利用側はこうなる。ヒュー、慣れててわかりやすいねぇ。
```cs
// 利用側: TipCalculator の具象型を直接知る
public class CheckoutService
{
    private readonly TipCalculator _tipCalculator = new();

    public Receipt Finalize(Order order)
    {
        var tip = _tipCalculator.CalculateTip(order.Subtotal, order.TipRate);
        return new Receipt(order, tip);
    }
}
```
🫥これでヨシ！結局やってること同じだよね？
え？「チップが常に%計算の米国式とは限らないだろ」「日本にも高級店にはサービス料がある」って？
そうだとしても、また**必要になったら**、そのときに 新しいTipCalculator を足せばいいだろ？今の顧客にはいないんだし。

「契約結合がモデル結合に変化」した…？何のことをおっしゃっているのだか？

## なんでわざわざ値をラップするわけ？
🫥レコード？とかいうのは、クラスの亜種で、なんか値を使うときに使うヤツらしい？でも、そもそも**値**ならプリミティブ型でよくない？
```cs
namespace Billing.Tipping;

public static class TipCalculator
{
    public static decimal CalculateTip(decimal subtotal, decimal rate)
        => subtotal * rate;
}
```
🫥ほら、これでMoneyとかTipRate とかいちいち宣言しなくて良くなりましたよ？俺ってやっぱ天才…？
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
🫥そして利用側はこうだね。
```cs
// 利用側: decimal の意味と 0~1 の制約を"暗黙的に"知っている必要がある
public class CheckoutService
{
    public Receipt Finalize(Order order)
    {
        var tipRate = 0.15m;
        var tip = TipCalculator.CalculateTip(order.SubtotalAmount, tipRate);
        return new Receipt(order, tip);
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
```cs
// 利用側: decimal の意味と 0~1 の制約を"暗黙的に"知っている必要がある
public class CheckoutService
{
    public Receipt Finalize(Order order)
    {
        var tipRate = 0.15m;
        var taxRate = 0.08m;
        var tip = TipCalculator.CalculateTip(order.SubtotalAmount, taxRate);
        // ↑ うっかり taxRate を渡しても型的にはまったく同じ。コンパイラは気づかない
        return new Receipt(order, tip);
    }
}
```

「モデル結合が機能結合に変化」した…？よくわかんないけど、難しい言葉でごまかそうとするの止めてもらえる？

## レートなんて DB から直接読めば DRY じゃん
🫥チップのレート 0.15 をベタ書きするのダサくない？聞いた話だと、Tax モジュールが `tax_rates` っていうレート管理テーブルを持ってるらしい。そっち直接読めばレート管理を一本化できて DRY じゃん！

```cs
namespace Billing.Tipping;

public static class TipCalculator
{
    public static decimal CalculateTipFromOrder(int orderId)
    {
        using var conn = new SqlConnection(ConnectionStrings.TaxModule);
        // Tax モジュールの内部テーブルに直接クエリ
        var subtotal = conn.QuerySingle<decimal>(
            "SELECT subtotal_before_tax FROM tax_module.orders WHERE order_id = @id",
            new { id = orderId });
        return subtotal * 0.15m;
    }
}
```
利用側もこんなにシンプル。
```cs
// 利用側: orderId さえ渡せばいい
var tip = TipCalculator.CalculateTipFromOrder(orderId);
```
🫥Order の中身を渡す必要もなくなって、利用側も超スリム。俺って天才過ぎない？
え…？「これはもはや侵入結合だ」…？「他モジュールの内部テーブル構造に依存するのは、そのモジュールの private なフィールドを覗き見してるのと同じ」って…？

「統合されている側のモジュールは、統合の存在すら知らない」…？なんでそんな悲しそうな顔するの？

---

そして、ある日、本番が静かに壊れました。

Tax チームが内部の migration で `subtotal_before_tax` カラムを `net_amount` にリネームしたのです。Tax モジュールのテストは全部通ります。なぜなら **Tax チームは、自分のテーブルを Billing.Tipping が直接見ていることを知らない** からです。Billing.Tipping のテストも、自分の DB を使っている限りは通ります。たいてい、こういう他チームのDB参照する場合の単体テストはモック使ってますからね。

---

ハッ…！私は一体何を、なんだかすごく頭が痛い…

# 何が起こっていたのか
茶番は差し置いて、YAGNI🫥の感覚はこうです。
- モジュールの契約を柔らかくしたい
- 早すぎる実装をしたくない
- とにかく全体を短くしたい

これは一概に否定すべき感覚ではないと思っています。
実際、彼の主張には **特定の条件下では正しいもの** がたくさんあります。問題は、彼女が一貫して「このコードは変わらない」「この要件は増えない」という ***低変動性を暗黙の前提*** としていることです。

Vlad Khononov の結合バランスモデルでは、ドメインを以下の3つに分類し、変動性の高さを判断します:

- **コアサブドメイン** — ビジネスの競争優位をもたらす領域。**高変動性**。競争優位性のために継続的に改善される
- **サポーティングサブドメイン** — 差別化要因ではないが必要なもの（管理画面の CRUD、ETL 等）。**低変動性**
- **汎用サブドメイン** — 既製品ソリューションがある領域（認証、決済等）。機能は安定、実装は差し替え候補

YAGNIちゃんの主張は、サポーティング領域であれば実用的な判断になり得ます。問題は、彼女が **すべての領域に同じ処方箋を当てる** こと、つまり、コアサブドメインでもサポーティングでも、同じ「シンプル」を適用してしまう点にあります。（彼の「シンプル」の理解が誤っていることは、Rich HickeyのSimple made Easyの講義をご覧いただくとわかりやすいです。）

| YAGNIの主張 | 正しい文脈 | 誤用の文脈 |
| --- | --- | --- |
| インターフェース不要 | ポリシー差し替えが想定されない領域 | コア、拡張や差し替えが継続的に起きる領域 |
| 値オブジェクト不要 | 単一文脈、型の取り違えリスクが低い | 複数種類のレート・単位が混在する領域 |
| 汎用化で DRY | 同じ *知識* の重複 | 形状が似た *別の意味* の処理 |
| 他モジュール DB を直読 | 同モジュール内の内部統合 | モジュール境界を越える統合 |

大事なのは、モジュールの変動性を定義し、それとバランスが取れた結合強度を知ることだと思います。
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
| 内部 DB 直読をやめ公開 API に戻す | 実装詳細を再カプセル化 | 侵入 → 機能 |
| `TipRate` / `Money` を復活 | 不変条件を型に押し込める | 機能 → モデル |
| `ITipCalculator` と複数実装を復活 | ポリシー差し替えを契約で吸収 | モデル → 契約 |

ちなみに、Vlad Khononovのモジュール結合バランスに関する考えは、以下のリポジトリでAIエージェント用のskillsにもなっています。こちらを一読するだけでも大変勉強になるかと思います。本と合わせて非常におすすめです。
https://github.com/vladikk/modularity