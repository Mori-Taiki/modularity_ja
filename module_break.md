こんにちは、エンジニアのT.Mです。今日は私の代わりに、私の中のYAGNI🫥なペルソナが記事を書いてくれるそうです。


YAGNI🫥：「強固なアーキテクチャって、堅苦しいよね？」「なんだか冗長に見えるし、何の意味があって分かれているのか、理解しがたいことばっかり！」「業務では絶対やれないこと、ブログではやっちゃおう！！！」

なんだか、気分が悪くなってきました。しかし、ここは温かい目で見守ることにします。

🫥「ここからは天才プログラマーであるところの俺様が、シンプルなコードってのを見せてやるよ」

# 堅苦しく、長いコード

🫥今回取り組むのはこのコード。飲食店の会計を行うCheckoutモジュールで、チップ込みの合計金額を計算するためのコードらしい。

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
```

そして、このモジュールは以下のように使われている。

```cs
// 利用側：ICheckoutWorkflow のみ知っている
public class OrderController(ICheckoutWorkflow checkout)
{
    public Receipt Submit(Order order) => checkout.Finalize(order);
}
```

🫥なんかクラスっぽいのがたくさんある。よく見るとインターフェースが2つもあるし、レコード？とか色々あってややこしい。
どうやら、やっていることは「購入金額 × チップ率」を足して合計を返すだけらしいが、**それをやるだけにこんなたくさんのコードが必要なのか？**
もっと"シンプル"にしてやろう。
なんか後ろから、「お前は"Simple"と"Easy"を混同している」という声が聞こえるが、無視して進めることにする。

（参考: [「Simple」と「Easy」はちがう。 - ログラスEngineers Blog](https://zenn.dev/loglass/articles/6b56dac587f02e)）

## このインターフェース、要る？

🫥 `CheckoutWorkflow` って結局のところ、中で `PercentageTipCalculator` を呼んでいるだけだ。利用側はインターフェース越しに一段噛まされて、実際の計算器に辿り着くまでに迷子になる。**一段減らせばシンプルだろ？**
何？これは「ファサード（窓口）パターンと言って、モジュールの玄関口を一つにして複雑さを減らすんだ」って？よくわからないけど`F2`で実装に飛べないのはちょっと…え？「選択して`Ctrl + F12`で飛べる」って？そんなのいちいち覚えてらんないよ…俺は"慣れた"手段で仕事がしたいんだ。

```cs
// Checkout モジュール：ファサード（ICheckoutWorkflow） を消し、計算器を public に"格上げ"
namespace Checkout;

public interface ITipCalculator
{
    Money Calculate(Money subtotal, TipRate rate);
}

public sealed class PercentageTipCalculator : ITipCalculator
{
    public Money Calculate(Money subtotal, TipRate rate) =>
        subtotal with { Amount = subtotal.Amount * rate.Value };
}
```

```cs
// 利用側：部品を直接知り、自分で組み立てる
public class OrderController(ITipCalculator tip)
{
    public Receipt Submit(Order order)
    {
        var tipAmount = tip.Calculate(order.Subtotal.Value, order.TipRate);
        var total     = order.Subtotal.Value with { Amount = order.Subtotal.Value.Amount + tipAmount.Amount };
        return new Receipt(order.Id, total);
    }
}
```

🫥これでヨシ！結局やってること同じだよね？1段減った分、スッキリしただろ？

え？「将来 `ICheckoutWorkflow` に税金計算や割引ロジックが入ってきたら、利用側もそれを自分で組み立て直さなきゃいけないだろ！」って？そうだとしても、また**必要になったら**、そのときに ファサード とやらを足せばいいだろ？今そんな要件ないんだし。

え…？「ファサード があった頃は、Checkout チームが内部を好きに動かしても利用側を壊さなかった」「インターフェースをはがしたことで、**内部構造**（どの計算者がどう並んでいるか）が利用側に漏出した」って…？

「契約結合がモデル結合に変化」した…？何のことをおっしゃっているのだか？

## なんでわざわざ値をラップするわけ？

🫥レコード？とかいうのは、クラスの亜種で、なんか値を使うときに使うヤツらしい？でも、そもそも**値**ならプリミティブ型でよくない？インターフェース経由で呼ぶのも面倒だし、ついでに static メソッドにしちまおう。

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

え？「レートが0〜1の範囲内かどうかのチェックはどこへ行ったんだ」って？ちゃんと計算側でチェックしてるでしょ。

え…？「`TipRate` と `TaxRate` が型で区別できなくなった」「取り違えてもコンパイラは黙って通す」…？そんなの変数名ちゃんとしてればそんなこと起こらないよ、人間を信じなって。

```cs
// 利用側：decimal の意味と 0〜1 の制約を"暗黙的に"知っている必要がある
public class OrderController
{
    public Receipt Submit(Order order)
    {
        var tipRate = 0.15m;
        var taxRate = 0.08m;
        var tip     = TipCalculator.Calculate(order.SubtotalAmount, taxRate);
        // ↑ うっかり taxRate を渡しても型的にはまったく同じ。コンパイラは気づかない
        var total   = Math.Round(order.SubtotalAmount + tip, 2);  // 丸め方針も自前
        return new Receipt(order.Id, new Money(total, "JPY"));
    }
}
```

🫥ほら、別にそんなの一回気をつければいいんだから、どうってことないでしょ？

---

…しばらく経って、返金（Refund）機能を作ることになった。返金でもサブトタル・チップ・丸めの扱いが必要だ。🫥「Checkout のロジックに似てるし、コピペでよくない？」

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

🫥コピペだから動作は同じ。完璧！

---

そしてある日、税制改正で「丸めは小数第0位（円単位）」に変わった。Checkout チームが `TipCalculator.Calculate` の `Math.Round(subtotal * rate, 2)` を `Math.Round(subtotal * rate, 0)` に直す。**でも Refunds 側は気づかない**。返金額と支払額の小数点以下が合わなくなり、会計が静かにズレはじめる。

え…？「"割引→税→チップの順で適用する"や"小数第2位で丸める"みたいな**ビジネスルールが複数の場所に重複**している」…？「仕様変更がすべてのコピーに追随しないと、会計が合わなくなる」…？

「モデル結合が機能結合に変化」した…？よくわかんないけど、難しい言葉でごまかそうとするの止めてもらえる？

## DB を直接見た方が速いじゃん

🫥そんなある日、上司から「Checkout モジュールが保留中のレシートを `PendingReceipts` テーブルに書き出しているから、それを Orders 側から読んで画面に出してくれ」と言われた。

🫥本来なら Checkout 側に「保留中レシートを取得する」API を生やすべきなんだろうが、面倒くさい。テーブル直接 SELECT した方が速いに決まってる。`internal` になってる？じゃあ `InternalsVisibleTo` で開けちゃえ。

```cs
// Checkout モジュール：「社内用」のヘルパーとテーブルを持っている
namespace Checkout;

[assembly: InternalsVisibleTo("Orders")]   // ← 穴をあけてしまった

public sealed class CheckoutWorkflow
{
    // 将来の最適化で差し替える予定の internal ヘルパー
    internal static decimal ApplyRoundingPolicy(decimal amount) => Math.Round(amount, 2);
}

// 本来は閉じているはずの内部 DbContext
internal class CheckoutDbContext : DbContext
{
    public DbSet<PendingReceiptRow> PendingReceipts { get; set; }
}

internal class PendingReceiptRow
{
    public Guid OrderId { get; set; }
    public decimal Subtotal { get; set; }
    public decimal Tip { get; set; }
}
```

```cs
// 利用側：Checkout の internal と DB スキーマを直接触る
namespace Orders;

public class OrderController(CheckoutDbContext checkoutDb)
{
    public Receipt Submit(Order order)
    {
        // Checkout モジュールの内部テーブルを直接 SELECT
        var pending = checkoutDb.PendingReceipts
                                .First(p => p.OrderId == order.Id.Value);

        // 内部の丸め方針ヘルパーを直接呼ぶ
        var total = CheckoutWorkflow.ApplyRoundingPolicy(pending.Subtotal + pending.Tip);

        return new Receipt(order.Id, new Money(total, "JPY"));
    }
}
```

🫥動いた！一行で済んだ！DRY！

---

…数週間後、Checkout チームが `PendingReceiptRow` の `Tip` カラムを `TipAmount` にリネームし、`ApplyRoundingPolicy` の第2引数に精度を取るようにシグネチャを拡張した。Checkout 単体のテストは全部緑。デプロイ。

そして Orders 側が本番で落ちる。**Checkout チームは、自分の internal が Orders から覗かれていることすら知らなかった。**

え…？「Checkout の"非公開"に利用側が依存したせいで、Checkout チームが internal を変更した瞬間に Orders がサイレントに壊れた」…？「コンパイラは通るが、"お客様の見積もりが突然合わなくなる"という形で本番事故化する」…？「一度これに陥ると、Checkout 側は自分のコードを動かすためにまず"他に誰が自分の internal を見ているか"を調査しなければならない」…？なんでそんな悲しそうな顔するの？

---

ハッ…！私は一体何を、なんだかすごく頭が痛い…

# 何が起こっていたのか

茶番は差し置いて、YAGNI🫥の感覚はこうです。

- モジュールの契約を柔らかくしたい
- 早すぎる実装をしたくない
- とにかく全体を短くしたい

これは一概に否定すべき感覚ではないと思っています。
実際、彼の主張には **特定の条件下では正しいもの** がたくさんあります。問題は、彼女が一貫して「このコードは変わらない」「この要件は増えない」という ***低変動性を暗黙の前提*** としていることです。

## 結合強度の4段階

YAGNI🫥の一手ずつ、コードは「より多くの知識」を利用側に漏らしていきました。Vlad Khononov の Balanced Coupling モデルでは、モジュール間で共有される知識の量によって、結合を4段階に分類します（弱い順）:

| 段階 | 利用側が Checkout について知っていること | 典型シグナル | 変更の波及 |
| --- | --- | --- | --- |
| **契約結合 (Contract)** | `Finalize(Order) → Receipt` のシグネチャだけ | Facade / DTO / Open-host service | 契約面を守れば内部は自由 |
| **モデル結合 (Model)** | 内部の計算者インターフェース群 | `internal` の interface が `public` に漏れる | 部品構成の変更が利用側に波及 |
| **機能結合 (Functional)** | 計算順序・丸め・不変条件などの**ビジネスルール** | 同じロジックが複数箇所にコピー | ルール変更が全コピーに追随しないと不整合 |
| **侵入結合 (Intrusive)** | 内部テーブル構造・`internal` メソッドの存在 | `InternalsVisibleTo`, 他モジュールの DB 直読 | 非公開の変更が**サイレントに**利用側を壊す |

上から下に下がるほど、**共有される知識の量が増え、モジュール境界が意味を失っていく**。契約結合は、利用側が知るべきことを「Facade が公開する契約面」だけに絞ることで、内部の変動性を外に漏らさない。

## 変動性とサブドメイン

ただし、結合の段階だけで設計の良し悪しは決まりません。Vlad Khononov のモデルで重要なのは、**結合の強さとその対象の変動性のバランス**です。

変動性の評価には、DDD の **サブドメイン** 分類が有用です:

- **コアサブドメイン** — ビジネスに競争優位をもたらすシステムの部分。組織が積極的に投資している「面白い問題」。**頻繁に変化する**。
- **サポーティングサブドメイン** — ビジネスが必要としているが差別化要因ではないもの（管理画面の CRUD、ETL など）。**めったに変わらない**。
- **ジェネリックサブドメイン** — 既製品ソリューションが存在する解決済みの問題（認証、決済、メール配信）。機能は成熟しているが、プロバイダーの差し替えが現実的な変更ベクターになる。

YAGNI🫥 の主張は、**サポーティング領域**であれば実用的な判断になり得ます。問題は、彼女が **すべての領域に同じ処方箋を当てる** こと。コアサブドメインにもサポーティングにも、同じ「シンプル」を適用してしまう点にあります。（彼の「シンプル」の理解が誤っていることは、Rich Hickey の Simple made Easy の講義をご覧いただくとわかりやすいです。）

| YAGNIの主張 | 正しい文脈 | 誤用の文脈 |
| --- | --- | --- |
| Facade 不要、直接呼べばいい | サポーティング領域、内部構成が当面固まっている | コア、内部構造が継続的に進化する領域 |
| 値オブジェクト不要、decimal で十分 | 単一文脈、型の取り違えリスクが低い | 複数種類のレート・単位が混在する領域 |
| DRY のためコピペ / internal 流用 | 本当にドメイン中立な処理 | 両モジュールが**意味**を持つ計算の共有 |

大事なのは、モジュールの変動性を定義し、それとバランスが取れた結合強度を選ぶことだと思います。
そして、それを一言で表現する用語を知っていれば、自分の中での認知負荷が低くなり、チームで共有されれば指摘もスムーズです。「これはモデル結合に降りているね」「ここは侵入結合になってるよ」と名前で呼べる設計議論は、感情論になりません。

# 下から読むと、リファクタリング

すでにお気づきの方もいらっしゃると思いますが、本ブログのモジュール結合についての定義は、Vlad Khononov の『ソフトウェア設計の結合バランス』に基づいています。

[ソフトウェア設計の結合バランス（Vlad Khononov）— Amazon](https://amzn.asia/d/02wEkr8p)

彼の定義では、モジュール結合は以下の4つの段階に分類されます。(実際にはコナーセンス等を使ってそれぞれの段階の内部でもレベルがあるのですが、簡単のために今回はそこには踏み込みませんでした。)

モジュール間の結合は、より弱い順から以下の通りです。

* 契約結合 (Contract coupling)
* モデル結合 (Model coupling)
* 機能結合 (Functional coupling)
* 侵入結合 (Intrusive coupling)

この記事では、あえてより疎である結合から、知識を漏出させていく形を取ってみました。
そのため、設計にこだわりのある方には読むのが苦痛の記事だったかもしれません。
この記事を下から読み直して、リファクタリングされていく気持ちよさを味わい、精神の安寧を得ていただければ幸いです。

| 下から読むと | 何が起きているか | 結合の移動 |
| --- | --- | --- |
| 内部 DB 直読と `InternalsVisibleTo` をやめ、公開 API に戻す | 実装詳細を再カプセル化 | 侵入 → 機能 |
| `TipRate` / `Money` / 丸め方針を型と責務に戻す | 不変条件とビジネスルールを型・1箇所に押し込める | 機能 → モデル |
| `ICheckoutWorkflow` Facade を復活し、計算器を `internal` に戻す | 内部構造の露出を契約で包み直す | モデル → 契約 |

# おわりに

ちなみに、Vlad Khononov のモジュール結合バランスに関する考えは、以下のリポジトリで AI エージェント用の skills にもなっています。こちらを一読するだけでも大変勉強になるかと思います。本と合わせて非常におすすめです。

[vladikk/modularity — GitHub](https://github.com/vladikk/modularity)
