こんにちは、エンジニアのT.Mです。本記事では Vlad Khononov 『ソフトウェア設計の結合バランス』に基づき、モジュール結合の4段階を Checkout 機能の同一仕様のコードで示します。

[:contents]

# 前提：仕様は変わらない

題材は飲食店の Checkout モジュールです。仕様は終始次の1つだけで、本記事を通じて一切変わりません。

- 注文の小計に対し、**割引 → 税** の順で適用する
- 各ステップで小数第2位に丸める
- 割引率・税率は `0〜1` の範囲

変わるのは「**そのルールを誰が知っているか**」だけです。利用側に漏れる知識が増えるほど、結合は強くなります。

# 結合の4段階

Khononov の定義では、モジュール結合は共有される知識の量で4段階に分かれます（弱い順）。

| 段階 | 利用側が知っていること | 変更の波及 |
| --- | --- | --- |
| 契約結合 | 公開された契約のシグネチャだけ | 契約を守れば内部は自由 |
| モデル結合 | 内部のドメインモデル | モデル変更が利用側に波及 |
| 機能結合 | ビジネスルール（順序・丸め・不変条件） | ルール変更が全箇所に追随しないと不整合 |
| 侵入結合 | 内部実装（テーブル・`internal` 等） | 非公開の変更がサイレントに利用側を壊す |

機能結合は内部にさらに段階があり、本記事では「制御結合」「重複した機能」の順で扱います（後者の方が強固です）。

以下、弱い順に同一仕様のコードを並べます。下から上に読み返すと、そのままリファクタリングの手順になります。

## 契約結合

利用側が知っているのは `CheckoutRequest` と `CheckoutResponse` の形だけです。内部のドメインモデル（`Order` / `Money` / `DiscountRate` / `TaxRate`）も、計算器の構成（`IDiscountCalculator` / `ITaxCalculator`）も、利用側からは見えません。

```cs
namespace Checkout;

// === 公開する契約：プリミティブ型のDTO ===
public sealed record CheckoutRequest(
    Guid OrderId, decimal Subtotal, decimal DiscountRate, decimal TaxRate, string Currency);
public sealed record CheckoutResponse(Guid OrderId, decimal Total, string Currency);

public interface ICheckoutService
{
    CheckoutResponse Checkout(CheckoutRequest request);
    IReadOnlyList<CheckoutResponse> GetRecent(int days);
}

// === 内部：業務領域を表現するモデル ===
internal sealed record Order(OrderId Id, Money Subtotal, DiscountRate Discount, TaxRate Tax);
internal sealed record Money(decimal Amount, string Currency);
internal readonly record struct DiscountRate
{
    public decimal Value { get; }
    public DiscountRate(decimal v)
    {
        if (v < 0m || v > 1m) throw new ArgumentOutOfRangeException(nameof(v));
        Value = v;
    }
}
internal readonly record struct TaxRate { /* 同上 */ }

internal sealed class CheckoutService(IDiscountCalculator d, ITaxCalculator t) : ICheckoutService
{
    public CheckoutResponse Checkout(CheckoutRequest req)
    {
        var order = new Order(
            new OrderId(req.OrderId),
            new Money(req.Subtotal, req.Currency),
            new DiscountRate(req.DiscountRate),
            new TaxRate(req.TaxRate));

        var afterDiscount = d.Apply(order.Subtotal, order.Discount);
        var total         = t.Apply(afterDiscount, order.Tax);

        return new CheckoutResponse(order.Id.Value, total.Amount, total.Currency);
    }
    public IReadOnlyList<CheckoutResponse> GetRecent(int days) { /* 略 */ }
}

internal interface IDiscountCalculator { Money Apply(Money m, DiscountRate r); }
internal interface ITaxCalculator      { Money Apply(Money m, TaxRate r); }

// === 利用側 ===
public class OrderController(ICheckoutService checkout)
{
    public CheckoutResponse Submit(Guid id, decimal sub, decimal disc, decimal tax)
        => checkout.Checkout(new CheckoutRequest(id, sub, disc, tax, "JPY"));
}
```

契約面さえ守れば、Checkout チームは内部のモデルも計算器の並びも自由に組み替えられます。

## モデル結合

「DTO への詰め替えが冗長」という理由で、内部モデルを `public` に格上げした状態です。利用側は `Order` / `Money` / `DiscountRate` / `TaxRate` という**内部のドメインモデルそのもの**を組み立てて渡すようになりました。

```cs
namespace Checkout;

// ドメインモデルを public に格上げ。DTOは消える
public sealed record Order(OrderId Id, Money Subtotal, DiscountRate Discount, TaxRate Tax);
public sealed record Receipt(OrderId Id, Money Total);
public sealed record Money(decimal Amount, string Currency);
public readonly record struct DiscountRate
{
    public decimal Value { get; }
    public DiscountRate(decimal v)
    {
        if (v < 0m || v > 1m) throw new ArgumentOutOfRangeException(nameof(v));
        Value = v;
    }
}
public readonly record struct TaxRate      { /* 同上 */ }

public interface ICheckoutService
{
    Receipt Checkout(Order order);
    IReadOnlyList<Receipt> GetRecent(int days);
}

public class OrderController(ICheckoutService checkout)
{
    public Receipt Submit(Guid id, decimal sub, decimal disc, decimal tax)
    {
        var order = new Order(
            new OrderId(id),
            new Money(sub, "JPY"),
            new DiscountRate(disc),   // ← 制約付き値オブジェクトの存在を知った
            new TaxRate(tax));
        return checkout.Checkout(order);
    }
}
```

利用側は内部モデルの構造変更（フィールド追加・名前変更・型変更）に追随する必要が生まれました。一方で、`DiscountRate` の `0〜1` 制約は **値オブジェクト自身**が保証しているため、不変条件は依然として1箇所にあります。

## 機能結合（制御）

「値オブジェクトをいちいち作るのが面倒」「割引も税も結局やってることは似ている」という発想で、ロジックを `decimal + bool` の汎用関数 `RateApplier` にまとめた状態です。値オブジェクトは消滅し、丸め方針は `RateApplier` の中に、適用順は利用側に分散しました。

```cs
namespace Checkout;

public static class RateApplier
{
    public static decimal Apply(decimal amount, decimal rate, bool subtract)
    {
        if (rate is < 0m or > 1m) throw new ArgumentOutOfRangeException();
        var delta = Math.Round(amount * rate, 2);
        return subtract ? amount - delta : amount + delta;
    }
}

public sealed record Order(Guid Id, decimal Subtotal, decimal DiscountRate, decimal TaxRate);
public sealed record Receipt(Guid OrderId, decimal Total);

public class OrderController
{
    public Receipt Submit(Order order)
    {
        var afterDiscount = RateApplier.Apply(order.Subtotal, order.DiscountRate, subtract: true);
        var total         = RateApplier.Apply(afterDiscount, order.TaxRate,       subtract: false);
        return new Receipt(order.Id, total);
    }
}
```

`bool subtract` のように **利用側に内部の分岐を選ばせる**結合を制御結合と呼びます。利用側はもはや「`DiscountRate` は引く」「`TaxRate` は足す」「順序は割引→税」というビジネスルールを自分で覚えていなければなりません。

## 機能結合（重複した機能）

「`bool` 引数で挙動が変わるのは読みづらい」「各 Controller の中で素直に書いた方が早い」と判断し、`RateApplier` をやめて各箇所にロジックを直書きした状態です。

```cs
public class OrderController
{
    public Receipt Submit(Order order)
    {
        var discount      = Math.Round(order.Subtotal * order.DiscountRate, 2);
        var afterDiscount = order.Subtotal - discount;
        var tax           = Math.Round(afterDiscount * order.TaxRate, 2);
        var total         = afterDiscount + tax;
        return new Receipt(order.Id, total);
    }
}

// 別の利用箇所（見積プレビュー、注文確認メール、管理画面の表示など）
public class OrderPreviewController
{
    public PreviewResult Preview(Order order)
    {
        // OrderControllerからコピペ。ガードは「プレビューだから不要」と判断して外した
        var discount      = Math.Round(order.Subtotal * order.DiscountRate, 2);
        var afterDiscount = order.Subtotal - discount;
        var tax           = Math.Round(afterDiscount * order.TaxRate, 2);
        var total         = afterDiscount + tax;
        return new PreviewResult(total);
    }
}
```

同じビジネスルールが複数箇所に**重複**しています。仕様変更時に全コピーへ反映できる保証はなく、コピー後に変数名を変えてしまえば grep でも追跡できません。Khononov はこれを機能結合の中でも特に強固な形と位置付けています（制御結合より強い）。制御結合では少なくとも分岐ロジック自体は1箇所にありましたが、ここでは**手続きそのもの**が散らばっています。

## 侵入結合

「Checkout チームに API を作ってもらうのを待っていられない」という理由で、Dashboard モジュールが Checkout の内部テーブルを直接 SELECT している状態です。

```cs
// Checkout内部
namespace Checkout;
internal class CheckoutDbContext : DbContext
{
    public DbSet<ReceiptRow> Receipts { get; set; }
}
internal class ReceiptRow
{
    public Guid OrderId { get; set; }
    public decimal Subtotal { get; set; }
    public decimal Discount { get; set; }
    public decimal Tax { get; set; }
    public decimal Total { get; set; }
    public DateTime CheckedOutAt { get; set; }
}

// Dashboardモジュール：Checkoutの "internal" を覗く
namespace Dashboard;

public class SalesDashboard(IDbConnection db)
{
    public decimal GetTaxCollectedThisMonth() =>
        db.ExecuteScalar<decimal>(@"
            SELECT SUM(Tax) FROM checkout.Receipts
            WHERE CheckedOutAt >= @from",
            new { from = DateTime.UtcNow.AddDays(-30) });
}
```

Checkout チームは `ReceiptRow` が外から参照されていることを**知りません**。カラム名の変更や別スキーマへの移動だけで、Dashboard はコンパイルが通ったまま本番でサイレントに壊れます。これを防ぐには「内部を直す前に他に誰が覗いているかを毎回調査する」必要が生じ、内部の自由が失われます。最も頑固な結合です。

# 下から読むとリファクタリング

ここまでを下から読み返すと、そのまま結合を弱める手順になります。

| 下から読むと | やっていること | 結合の移動 |
| --- | --- | --- |
| Dashboard が Checkout の内部DBを直読する → 公開APIを要求する | 内部実装への侵入を契約の向こう側に戻す | 侵入結合 → 機能結合（重複） |
| 各Controllerにコピペされたロジックを Checkout に集約する | 散らばった手続きを1箇所に集める | 機能結合（重複） → 機能結合（制御） |
| `RateApplier(bool)` を解体し、`DiscountRate` / `TaxRate` / `Money` を復活させる | 不変条件・丸め・適用順を型と1箇所に押し込める | 機能結合（制御） → モデル結合 |
| 内部モデルを `internal` に戻し、公開DTOを設ける | 内部構造の露出を契約で包む | モデル結合 → 契約結合 |

# 変動性とのバランス

ここまで「弱い結合ほど良い」という前提で書きましたが、実務ではそうとは限りません。Khononov のモデルでは、結合と並んで**変動性（Volatility）** が重要な軸になります。

- 変動性が低い領域（事実上変わらないコード）では、強い結合でも実害はほぼ出ません。
- 変動性が高い領域（コアサブドメインなど）では、弱い結合でも継続的なツラみになります。

契約・値オブジェクト・Facade はいずれも「将来の変更を、利用側に漏らさずに内側で吸収する」ための投資です。**Checkout がビジネスの中核なら投資する価値があり、補助的な役割なら過剰**になり得る、という判断が常に必要です。

なお、本来はもう一つ「**距離**（同一パッケージ／別ライブラリ／同期／非同期／同一チーム／別チーム）」という軸もありますが、ここでは紙面の都合で割愛します。詳しくは原著をお読みください。

[ソフトウェア設計の結合バランス（Vlad Khononov）— Amazon](https://amzn.asia/d/02wEkr8p)

## おまけ

Vlad Khononov のモジュール結合バランスに関する考えは、AI エージェント向けの skills としても公開されています。

[vladikk/modularity — GitHub](https://github.com/vladikk/modularity)

# おわりに
KENTEMでは、様々な拠点でエンジニアを大募集しています！
建設×ITにご興味頂いた方は、是非下記のリンクからご応募ください。
[https://recruit.kentem.jp:embed:cite]
[https://career.kentem.jp:embed:cite]
