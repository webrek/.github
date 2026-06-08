# Webrek Laravel Packages

A small collection of focused, production-minded Laravel packages. Each one does
one thing well, ships with tests, static analysis and CI, and targets **Laravel
12 / PHP 8.2+**.

| Package | What it does |
| --- | --- |
| [**laravel-idempotency**](https://github.com/webrek/laravel-idempotency) | Safe request retries via the `Idempotency-Key` header. |
| [**laravel-money**](https://github.com/webrek/laravel-money) | An immutable money value object with exact arithmetic. |
| [**laravel-state-machine**](https://github.com/webrek/laravel-state-machine) | Declarative state machines for Eloquent models. |
| [**laravel-feature-flags**](https://github.com/webrek/laravel-feature-flags) | Feature flags with rollouts, targeting and A/B variants. |
| [**laravel-health-ui**](https://github.com/webrek/laravel-health-ui) | A production health dashboard and JSON endpoint. |

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

---

## Using them together

A single order flow touches all five — safe to retry, exact money, a guarded
lifecycle, a gated feature, and observable health:

```php
// routes/web.php
Route::post('/orders', StoreOrderController::class)->middleware('idempotency');

// StoreOrderController
public function __invoke(Request $request)
{
    $total = Money::of($request->input('amount'), 'MXN');
    $total = $total->plus($total->percentage(16));            // money

    $order = Order::create(['total' => $total]);              // state seeded to "pending"

    if (Features::active('instant-capture', $request->user())) {  // feature-flags
        $order->stateMachine()->apply('pay');                // state-machine (atomic)
    }

    return response()->json($order, 201);                    // idempotency replays on retry
}
```

## Quality

Every package ships with:

- A test suite (PHPUnit + Orchestra Testbench)
- Static analysis at **PHPStan level 6** (Larastan)
- **Laravel Pint** formatting (Laravel preset)
- GitHub Actions CI across PHP 8.2, 8.3 and 8.4

## License

All packages are released under the [MIT license](https://opensource.org/licenses/MIT).
