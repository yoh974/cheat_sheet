Top, on passe au **commerce / e-commerce**.
Voici un pack de **snippets PHP/Symfony avancés** prêts à l’emploi : pricing, TVA, promos, stock, paiements, webhooks, etc. (Commentaires en **anglais**.)

---

## 1) Money & VAT (net ↔ gross with rounding rules)

```php
final readonly class Money
{
    public function __construct(public int $cents, public string $currency = 'EUR') {
        if ($cents < 0) throw new \InvalidArgumentException('Negative money');
    }
    public static function fromFloat(float $v, string $cur='EUR'): self { return new self((int) round($v*100), $cur); }
    public function asFloat(): float { return $this->cents / 100; }
    public function add(self $o): self { self::assertCur($this,$o); return new self($this->cents+$o->cents,$this->currency); }
    public function mul(float $k): self { return new self((int) round($this->cents*$k), $this->currency); }
    private static function assertCur(self $a,self $b): void { if($a->currency!==$b->currency) throw new \DomainException('Currency mismatch'); }
}

final class Vat
{
    /** Convert net → gross using VAT percentage, round at 2 decimals */
    public static function netToGross(Money $net, float $vatRate): Money {
        return Money::fromFloat($net->asFloat() * (1 + $vatRate));
    }
    /** Split gross into net + vat */
    public static function splitGross(Money $gross, float $vatRate): array {
        $netFloat = $gross->asFloat() / (1 + $vatRate);
        $net = Money::fromFloat($netFloat, $gross->currency);
        $vat = Money::fromFloat($gross->asFloat() - $net->asFloat(), $gross->currency);
        return [$net, $vat];
    }
}
```

---

## 2) Cart totals with discounts (stackable promotions)

```php
final class CartTotals
{
    /** @param array<int,array{price:Money, qty:int}> $lines */
    public static function compute(array $lines, array $discounts): array
    {
        $subtotal = array_reduce($lines, fn(Money $acc, $l) => $acc->add($l['price']->mul($l['qty'])), new Money(0));
        // Apply stackable discounts: fixed first, then percentage
        $fixed = array_sum(array_map(fn($d)=>$d['type']==='fixed' ? $d['amount'] : 0, $discounts));
        $percent = array_reduce($discounts, fn($p,$d)=>$p+($d['type']==='percent'?$d['amount']:0), 0.0);
        $afterFixed = max(0, $subtotal->cents - (int)$fixed);
        $afterPercent = (int) round($afterFixed * (1 - $percent));
        return [
            'subtotal' => $subtotal,
            'discount' => new Money($subtotal->cents - $afterPercent),
            'total_excl_vat' => new Money($afterPercent),
        ];
    }
}
```

---

## 3) Promotion rule engine (specification-style)

```php
interface PromotionRule { public function isEligible(array $context): bool; }

final class MinCartAmountRule implements PromotionRule {
    public function __construct(private Money $min) {}
    public function isEligible(array $ctx): bool { return ($ctx['subtotal']->cents ?? 0) >= $this->min->cents; }
}

final class CategoryContainsRule implements PromotionRule {
    public function __construct(private string $categoryId) {}
    public function isEligible(array $ctx): bool { return in_array($this->categoryId, $ctx['categories'] ?? [], true); }
}

final class AndRule implements PromotionRule {
    public function __construct(private PromotionRule ...$rules) {}
    public function isEligible(array $ctx): bool { foreach($this->rules as $r) if(!$r->isEligible($ctx)) return false; return true; }
}
```

---

## 4) Stock reservation (pessimistic lock) for checkout

```php
final class InventoryService
{
    public function __construct(private \Doctrine\ORM\EntityManagerInterface $em) {}

    /** Reserve quantity atomically to avoid overselling */
    public function reserve(int $skuId, int $qty): void
    {
        $q = $this->em->createQuery('SELECT i FROM App\Entity\Inventory i WHERE i.sku = :sku')
            ->setParameter('sku', $skuId)
            ->setLockMode(\Doctrine\DBAL\LockMode::PESSIMISTIC_WRITE);
        $inv = $q->getSingleResult(); // row is locked
        if ($inv->available < $qty) throw new \RuntimeException('Out of stock');
        $inv->available -= $qty; $inv->reserved += $qty;
    }
}
```

---

## 5) Idempotent checkout (avoid duplicate orders)

```php
#[Route('/checkout', methods:['POST'])]
public function checkout(Request $r, EntityManagerInterface $em): JsonResponse
{
    $key = $r->headers->get('Idempotency-Key'); // client must send one
    if (!$key) return $this->json(['error'=>'Missing Idempotency-Key'], 400);

    $conn = $em->getConnection();
    $n = $conn->executeStatement('INSERT IGNORE INTO idempotency_keys (k, created_at) VALUES (?, NOW())', [$key]);
    if ($n === 0) { // key already used
        return $this->json(['status'=>'duplicate'], 200);
    }

    // ... create order, capture payment, commit
    return $this->json(['status'=>'ok'], 201);
}
```

---

## 6) Payment Port/Adapter (Stripe/Adyen swappable)

```php
interface PaymentGateway {
    public function authorize(Money $amount, string $orderId, array $meta=[]): string; // returns authId
    public function capture(string $authId, ?Money $amount=null): void;
    public function refund(string $paymentId, Money $amount): void;
}

final class StripeGateway implements PaymentGateway {
    public function __construct(private \Stripe\StripeClient $client) {}
    public function authorize(Money $amount, string $orderId, array $meta=[]): string {
        $pi = $this->client->paymentIntents->create([
            'amount'=>$amount->cents, 'currency'=>$amount->currency, 'capture_method'=>'manual', 'metadata'=>$meta+['orderId'=>$orderId]
        ]);
        return $pi->id;
    }
    public function capture(string $authId, ?Money $amount=null): void {
        $this->client->paymentIntents->capture($authId, $amount?['amount_to_capture'=>$amount->cents]:[]);
    }
    public function refund(string $paymentId, Money $amount): void {
        $this->client->refunds->create(['payment_intent'=>$paymentId, 'amount'=>$amount->cents]);
    }
}
```

---

## 7) Fraud score (lightweight, rule-based + signals)

```php
final class FraudScorer
{
    /** Return score 0..100; >70 = block, 40..70 = manual review */
    public function score(array $signals): int
    {
        $score = 0;
        if (($signals['email_domain_free'] ?? false) && ($signals['order_amount'] ?? 0) > 30000) $score += 25;
        if (($signals['ip_country'] ?? '') !== ($signals['billing_country'] ?? '')) $score += 20;
        if (($signals['failed_attempts'] ?? 0) >= 3) $score += 30;
        if (($signals['shipping_to_locker'] ?? false)) $score += 15;
        return min(100, $score);
    }
}
```

---

## 8) Coupon validation (single-use, expiry, combinability)

```php
final class CouponValidator
{
    public function validate(array $coupon, array $cartCtx): void
    {
        if (!($coupon['active'] ?? false)) throw new \DomainException('Coupon disabled');
        if (isset($coupon['expires_at']) && new \DateTimeImmutable($coupon['expires_at']) < new \DateTimeImmutable()) {
            throw new \DomainException('Coupon expired');
        }
        if (($coupon['single_use'] ?? false) && ($coupon['already_used'] ?? false)) {
            throw new \DomainException('Coupon already used');
        }
        if (!empty($cartCtx['has_other_coupons']) && !($coupon['combinable'] ?? false)) {
            throw new \DomainException('Not combinable');
        }
    }
}
```

---

## 9) Shipping rate selection (zones, weight, fallback)

```php
final class ShippingRates
{
    /** @param array{country:string,weight_g:int,subtotal:Money} $ctx */
    public static function best(array $ctx): array
    {
        $zones = [
            'FR' => [['max_g'=>500, 'price'=>Money::fromFloat(4.9)], ['max_g'=>2000, 'price'=>Money::fromFloat(7.9)]],
            'EU' => [['max_g'=>500, 'price'=>Money::fromFloat(8.9)], ['max_g'=>2000, 'price'=>Money::fromFloat(14.9)]],
        ];
        $zone = $zones[$ctx['country']] ?? $zones['EU'];
        foreach ($zone as $rate) if ($ctx['weight_g'] <= $rate['max_g']) return ['price'=>$rate['price'], 'service'=>'Standard'];
        return ['price'=>Money::fromFloat(24.9), 'service'=>'Heavy']; // fallback
    }
}
```

---

## 10) VAT number (EU VIES) format pre-check

```php
final class VatNumber
{
    /** Basic syntactic check; real validation via VIES SOAP/REST is recommended */
    public static function isValidFormat(string $vat): bool
    {
        $vat = strtoupper(preg_replace('/\s+/', '', $vat));
        return (bool) preg_match('/^[A-Z]{2}[A-Z0-9]{8,12}$/', $vat);
    }
}
```

---

## 11) Order number generator (prefix + yymmdd + seq)

```php
final class OrderNumberGenerator
{
    public function __construct(private \Doctrine\DBAL\Connection $db) {}

    public function next(): string
    {
        $date = (new \DateTimeImmutable())->format('ymd');
        $this->db->executeStatement('INSERT INTO order_seq (seq_date) VALUES (?) ON DUPLICATE KEY UPDATE counter=LAST_INSERT_ID(counter+1)', [$date]);
        $seq = (int) $this->db->lastInsertId();
        return sprintf('ORD-%s-%05d', $date, $seq);
    }
}
```

---

## 12) PSP webhook (signature verify + replay guard)

```php
#[Route('/webhooks/psp', methods:['POST'])]
public function psp(Request $r, LoggerInterface $log, EntityManagerInterface $em): JsonResponse
{
    $payload = $r->getContent();
    $sig = $r->headers->get('X-Signature') ?? '';
    $secret = $_ENV['PSP_WEBHOOK_SECRET'];

    // constant-time compare
    $expected = hash_hmac('sha256', $payload, $secret);
    if (!hash_equals($expected, $sig)) { $log->warning('Bad signature'); return $this->json([], 400); }

    $event = json_decode($payload, true, 512, JSON_THROW_ON_ERROR);
    $id = $event['id'] ?? null;

    // replay guard
    $ins = $em->getConnection()->executeStatement('INSERT IGNORE INTO webhook_events(id, received_at) VALUES (?, NOW())', [$id]);
    if ($ins === 0) return $this->json(['status'=>'duplicate'], 200);

    // handle event type (payment.succeeded, refund.created, ...)
    // ...
    return $this->json(['status'=>'ok'], 200);
}
```

---

## 13) Search & filters (safe sorting whitelist)

```php
$allowedSort = ['createdAt','price','name'];
$sort  = in_array($req->query->get('sort'), $allowedSort, true) ? $req->query->get('sort') : 'createdAt';
$order = $req->query->get('order') === 'asc' ? 'ASC' : 'DESC';

$qb = $repo->createQueryBuilder('p')
    ->andWhere(':q IS NULL OR p.name LIKE :q')->setParameter('q', $q ? "%$q%" : null)
    ->andWhere(':cat IS NULL OR p.category = :cat')->setParameter('cat', $cat ?: null)
    ->orderBy('p.'.$sort, $order);
```

---

## 14) Multi-currency FX with cache & timestamp

```php
final class Fx
{
    public function __construct(private array $rates, private string $base='EUR', private \DateTimeImmutable $asOf = new \DateTimeImmutable()) {}
    public function convert(Money $m, string $to): Money
    {
        if ($m->currency === $to) return $m;
        $eur = $m->asFloat() / ($this->rates[$m->currency] ?? 1.0);
        return Money::fromFloat($eur * ($this->rates[$to] ?? 1.0), $to);
    }
    public function asOf(): \DateTimeImmutable { return $this->asOf; }
}
```

---

## 15) SKU builder (options → deterministic SKU)

```php
final class SkuBuilder
{
    /** @param array<string,string> $options e.g. ['color'=>'BLK','size'=>'L'] */
    public static function build(string $base, array $options): string
    {
        ksort($options); // deterministic
        $suffix = strtoupper(implode('-', array_map(fn($k,$v)=>"$k:$v", array_keys($options), $options)));
        return strtoupper($base.'-'.$suffix);
    }
}
```

---

## 16) Soft capture then partial refund (post-shipping adjustment)

```php
final class CaptureService
{
    public function __construct(private PaymentGateway $psp) {}

    /** Capture at ship time, refund difference if final amount < authorized */
    public function settle(string $authId, Money $finalAmount, Money $authorized): void
    {
        $this->psp->capture($authId, $finalAmount);
        if ($finalAmount->cents < $authorized->cents) {
            $this->psp->refund($authId, new Money($authorized->cents - $finalAmount->cents, $authorized->currency));
        }
    }
}
```

---

