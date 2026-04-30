

## 契約結合

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

## モデル結合
モデル結合

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

## 機能結合（制御）

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

## 機能結合（重複した機能）

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

## 侵入結合

```cs
// Checkout内部（段階0からずっと存在）
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