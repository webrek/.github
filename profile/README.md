# Webrek Laravel Packages

A small collection of focused, production-minded Laravel packages. Each one does
one thing well, ships with tests, static analysis and CI, and targets **Laravel
12 & 13 / PHP 8.2+**.

| Package | What it does |
| --- | --- |
| [**laravel-idempotency**](https://github.com/webrek/laravel-idempotency) | Safe request retries via the `Idempotency-Key` header. |
| [**laravel-money**](https://github.com/webrek/laravel-money) | An immutable money value object with exact arithmetic. |
| [**laravel-state-machine**](https://github.com/webrek/laravel-state-machine) | Declarative state machines for Eloquent models. |
| [**laravel-feature-flags**](https://github.com/webrek/laravel-feature-flags) | Feature flags with rollouts, targeting and A/B variants. |
| [**laravel-health-ui**](https://github.com/webrek/laravel-health-ui) | A production health dashboard and JSON endpoint. |
| [**laravel-outbox**](https://github.com/webrek/laravel-outbox) | A transactional outbox for reliable, atomic message delivery. |
| [**laravel-circuit-breaker**](https://github.com/webrek/laravel-circuit-breaker) | Fail fast when a dependency is down, and recover automatically. |
| [**laravel-data-retention**](https://github.com/webrek/laravel-data-retention) | Keep records for a window, then delete or anonymize them automatically. |
| [**laravel-mx-validation**](https://github.com/webrek/laravel-mx-validation) | Validate Mexican identifiers (RFC, CURP, CLABE, NSS, CP) with real check digits. |

---

## laravel-idempotency

Make write requests safe to retry. A client sends a unique `Idempotency-Key`; a
repeat of the same request replays the original response instead of running
twice. Concurrent duplicates are serialised with an atomic lock, and a key reused
with a different payload is rejected.

```php
Route::post('/orders', [OrderController::class, 'store'])->middleware('idempotency');
```

Cache-backed, no migrations. Fires an `IdempotentReplay` event and supports
per-route TTLs.

```bash
composer require webrek/laravel-idempotency
```

## laravel-money

Money done right: stored as integer minor units, exact integer arithmetic, and
rounding only where you ask for it.

```php
$price = Money::of('19.99', 'USD');
$total = $price->plus($price->percentage(16));   // + 16% tax
$total->format();                                 // "USD 23.19"

Money::ofMinor(100, 'USD')->split(3);             // [0.34, 0.33, 0.33] — no cent lost
$orders->sumMoney('total');                        // sum a collection
$price->convert('EUR', $rates);                    // currency conversion
```

Eloquent casts, a validation rule, allocation, conversion and locale formatting.

```bash
composer require webrek/laravel-money
```

## laravel-state-machine

Declare the states a model can be in and the transitions between them; the
package enforces them with guards, events, optional history and atomic effects.

```php
'ship' => Transition::from('paid')->to('shipped')
    ->guard(fn ($order) => filled($order->address))
    ->using(fn ($order) => $order->warehouse->reserve()),   // atomic with the state change
```

```php
$order->stateMachine()->apply('ship');
$order->stateMachine()->toMermaid();   // render a diagram
```

```bash
composer require webrek/laravel-state-machine
```

## laravel-feature-flags

Ship features gradually with deterministic percentage rollouts, rule-based
targeting and A/B variants — flip them at runtime from a built-in dashboard, no
deploy.

```php
Features::create('new-checkout', rollout: 25);
Features::active('new-checkout', $user);
Features::variant('button-color', $user);   // 'blue' | 'green'
```

```blade
@feature('new-checkout') <x-checkout.v2 /> @endfeature
```

Database or array store, a `@feature` directive, `feature` middleware, artisan
commands and a web dashboard at `/feature-flags`.

```bash
composer require webrek/laravel-feature-flags
```

## laravel-health-ui

Real health checks (database, cache, disk, queue, scheduler, migrations, TLS
certs, external HTTP) behind one route — JSON `200`/`503` for uptime monitors and
a status page for humans.

```bash
curl -H "Accept: application/json" https://your-app.test/health
php artisan health:check    # exits non-zero when unhealthy
```

Pluggable: implement the `Check` contract and register your own.

```bash
composer require webrek/laravel-health-ui
```

## laravel-outbox

Stop losing events to dual writes. Stage a message inside the same database
transaction as your business write — they commit together, so a rolled-back
change never fires an event and a committed one never loses one — and a relay
delivers it afterwards with retries and exponential backoff.

```php
DB::transaction(function () use ($order) {
    $order->markPaid();
    Outbox::publish('order.paid', ['order_id' => $order->id]);   // atomic with the write
});
```

A pluggable publisher (events by default), multi-worker-safe claiming, automatic
reclaim of stuck messages, and `outbox:work` / `outbox:prune` commands. The
producer half of exactly-once — pair it with `laravel-idempotency` on the
consumer.

```bash
composer require webrek/laravel-outbox
```

## laravel-circuit-breaker

Stop a failing dependency from taking your app down with it. After enough
failures the circuit trips open and fails fast — no more requests piling up on a
dead endpoint — then probes for recovery and closes itself when the service is
healthy again.

```php
$response = CircuitBreaker::for('payments')->call(
    fn () => Http::timeout(3)->post($url, $payload)->throw(),
    fallback: fn () => null,   // returned while the circuit is open
);
```

Three states (closed → open → half-open), distributed state in the cache,
per-circuit thresholds, ignored exceptions and lifecycle events.

```bash
composer require webrek/laravel-circuit-breaker
```

## laravel-data-retention

Keep personal data only as long as you should. Declare a retention window per
model and what happens when rows age out — delete or anonymize them — and a
scheduled command enforces it, logging every row it touches for your compliance
trail.

```php
public function retentionPolicy(RetentionPolicy $policy): RetentionPolicy
{
    return $policy
        ->since('last_seen_at')->keepFor(365)            // a year of inactivity…
        ->where(fn ($q) => $q->where('legal_hold', false))
        ->anonymize([                                    // …then scrub the PII
            'name'  => '[redacted]',
            'email' => fn ($c) => "anon+{$c->id}@example.test",
        ], markColumn: 'anonymized_at');
}
```

`delete()`, `forceDelete()` and `anonymize()` actions, legal-hold scoping, a
`data_retention_log` audit table, and `retention:run` / `retention:list`
commands.

```bash
composer require webrek/laravel-data-retention
```

## laravel-mx-validation

Validate the Mexican identifiers your forms collect — RFC, CURP, CLABE, NSS and
código postal — with real check-digit verification, not just a regex.

```php
$request->validate([
    'rfc'   => ['required', 'rfc'],     // structure + date + check digit
    'curp'  => ['required', 'curp'],
    'clabe' => ['required', 'clabe'],
]);
```

```php
use Webrek\MxValidation\ValueObjects\Curp;

$curp = Curp::parse('PEPJ900101HDFRRN09');
$curp->stateName();   // "Ciudad de México"
$curp->birthDate();   // Carbon 1990-01-01
```

Value objects, Eloquent casts and a Faker provider for valid sample data.

```bash
composer require webrek/laravel-mx-validation
```

---

## Using them together

A single order flow touches seven of them — safe to retry, exact money, a guarded
lifecycle, a gated feature, a payment shielded by a circuit breaker, a reliably
published event, and observable health:

```php
// routes/web.php
Route::post('/orders', StoreOrderController::class)->middleware('idempotency');

// StoreOrderController
public function __invoke(Request $request)
{
    $total = Money::of($request->input('amount'), 'MXN');
    $total = $total->plus($total->percentage(16));                // money

    $order = DB::transaction(function () use ($request, $total) {
        $order = Order::create(['total' => $total]);              // state seeded to "pending"

        if (Features::active('instant-capture', $request->user())) {  // feature-flags
            CircuitBreaker::for('payments')->call(               // circuit-breaker
                fn () => $order->stateMachine()->apply('pay'),  // state-machine (atomic)
            );
        }

        Outbox::publish('order.placed', ['id' => $order->id]);   // outbox: commits with the order
        return $order;
    });

    return response()->json($order, 201);                        // idempotency replays on retry
}
```

## Quality

Every package ships with:

- A test suite (PHPUnit + Orchestra Testbench)
- Static analysis at **PHPStan level 6** (Larastan)
- **Laravel Pint** formatting (Laravel preset)
- GitHub Actions CI across PHP 8.2 through 8.5 (Laravel 12 and 13)

## License

All packages are released under the [MIT license](https://opensource.org/licenses/MIT).
