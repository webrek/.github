# Paquetes Laravel de Webrek

Una pequeña colección de paquetes de Laravel enfocados y pensados para
producción. Cada uno hace una sola cosa bien, viene con pruebas, análisis
estático y CI, y apunta a **Laravel 12 y 13 / PHP 8.2+**.

| Paquete | Qué hace |
| --- | --- |
| [**laravel-idempotency**](https://github.com/webrek/laravel-idempotency) | Reintentos seguros de peticiones mediante el encabezado `Idempotency-Key`. |
| [**laravel-money**](https://github.com/webrek/laravel-money) | Un objeto de valor de dinero inmutable con aritmética exacta. |
| [**laravel-state-machine**](https://github.com/webrek/laravel-state-machine) | Máquinas de estados declarativas para modelos Eloquent. |
| [**laravel-feature-flags**](https://github.com/webrek/laravel-feature-flags) | Feature flags con rollouts, segmentación y variantes A/B. |
| [**laravel-health-ui**](https://github.com/webrek/laravel-health-ui) | Un panel de salud para producción y un endpoint JSON. |
| [**laravel-outbox**](https://github.com/webrek/laravel-outbox) | Un transactional outbox para entrega de mensajes confiable y atómica. |
| [**laravel-circuit-breaker**](https://github.com/webrek/laravel-circuit-breaker) | Falla rápido cuando una dependencia está caída y se recupera automáticamente. |
| [**laravel-data-retention**](https://github.com/webrek/laravel-data-retention) | Conserva registros durante un periodo y luego los elimina o anonimiza automáticamente. |
| [**laravel-mx-validation**](https://github.com/webrek/laravel-mx-validation) | Valida identificadores mexicanos (RFC, CURP, CLABE, NSS, CP) con dígitos verificadores reales. |
| [**arco**](https://github.com/webrek/arco) | Solicitudes ARCO de titulares de datos y un registro de consentimiento (LFPDPPP) — agnóstico al framework, con un puente para Laravel. |
| [**sat-69b**](https://github.com/webrek/sat-69b) | Verifica RFCs contra la lista 69-B del SAT (EFOS/EDOS) — núcleo PHP, con puente para Laravel. |
| [**cfdi**](https://github.com/webrek/cfdi) | Construye, sella y timbra CFDI 4.0 sobre phpcfdi — núcleo PHP, con puente para Laravel y driver de PAC. |

---

## laravel-idempotency

Haz que las peticiones de escritura sean seguras de reintentar. El cliente envía
un `Idempotency-Key` único; una repetición de la misma petición reproduce la
respuesta original en lugar de ejecutarse dos veces. Los duplicados concurrentes
se serializan con un lock atómico, y una clave reutilizada con un payload
distinto se rechaza.

```php
Route::post('/orders', [OrderController::class, 'store'])->middleware('idempotency');
```

Respaldado por caché, sin migraciones. Dispara un evento `IdempotentReplay` y
admite TTLs por ruta.

```bash
composer require webrek/laravel-idempotency
```

## laravel-money

Dinero hecho bien: almacenado como unidades menores enteras, aritmética entera
exacta y redondeo solo donde tú lo pides.

```php
$price = Money::of('19.99', 'USD');
$total = $price->plus($price->percentage(16));   // + 16% de impuesto
$total->format();                                 // "USD 23.19"

Money::ofMinor(100, 'USD')->split(3);             // [0.34, 0.33, 0.33] — sin perder un centavo
$orders->sumMoney('total');                        // suma una colección
$price->convert('EUR', $rates);                    // conversión de divisas
```

Casts de Eloquent, una regla de validación, asignación, conversión y formateo
según locale.

```bash
composer require webrek/laravel-money
```

## laravel-state-machine

Declara los estados en los que puede estar un modelo y las transiciones entre
ellos; el paquete las hace cumplir con guards, eventos, historial opcional y
efectos atómicos.

```php
'ship' => Transition::from('paid')->to('shipped')
    ->guard(fn ($order) => filled($order->address))
    ->using(fn ($order) => $order->warehouse->reserve()),   // atómico con el cambio de estado
```

```php
$order->stateMachine()->apply('ship');
$order->stateMachine()->toMermaid();   // renderiza un diagrama
```

```bash
composer require webrek/laravel-state-machine
```

## laravel-feature-flags

Lanza funcionalidades de forma gradual con rollouts por porcentaje
deterministas, segmentación basada en reglas y variantes A/B — actívalas en
tiempo de ejecución desde un panel integrado, sin necesidad de un deploy.

```php
Features::create('new-checkout', rollout: 25);
Features::active('new-checkout', $user);
Features::variant('button-color', $user);   // 'blue' | 'green'
```

```blade
@feature('new-checkout') <x-checkout.v2 /> @endfeature
```

Almacenamiento en base de datos o en arreglo, una directiva `@feature`, un
middleware `feature`, comandos de artisan y un panel web en `/feature-flags`.

```bash
composer require webrek/laravel-feature-flags
```

## laravel-health-ui

Verificaciones de salud reales (base de datos, caché, disco, cola, scheduler,
migraciones, certificados TLS, HTTP externo) detrás de una sola ruta — JSON
`200`/`503` para monitores de uptime y una página de estado para humanos.

```bash
curl -H "Accept: application/json" https://your-app.test/health
php artisan health:check    # sale con código distinto de cero cuando no está sano
```

Extensible: implementa el contrato `Check` y registra el tuyo.

```bash
composer require webrek/laravel-health-ui
```

## laravel-outbox

Deja de perder eventos por dual writes. Coloca un mensaje dentro de la misma
transacción de base de datos que tu escritura de negocio — se confirman juntos,
de modo que un cambio revertido nunca dispara un evento y uno confirmado nunca
pierde ninguno — y un relay lo entrega después con reintentos y backoff
exponencial.

```php
DB::transaction(function () use ($order) {
    $order->markPaid();
    Outbox::publish('order.paid', ['order_id' => $order->id]);   // atómico con la escritura
});
```

Un publisher extensible (eventos por defecto), claiming seguro para múltiples
workers, recuperación automática de mensajes atascados y comandos `outbox:work`
/ `outbox:prune`. La mitad productora del exactly-once — combínalo con
`laravel-idempotency` en el consumidor.

```bash
composer require webrek/laravel-outbox
```

## laravel-circuit-breaker

Evita que una dependencia que falla arrastre a tu aplicación consigo. Tras
suficientes fallos el circuito se abre y falla rápido — sin más peticiones
acumulándose sobre un endpoint muerto — luego sondea para detectar la
recuperación y se cierra solo cuando el servicio vuelve a estar sano.

```php
$response = CircuitBreaker::for('payments')->call(
    fn () => Http::timeout(3)->post($url, $payload)->throw(),
    fallback: fn () => null,   // se devuelve mientras el circuito está abierto
);
```

Tres estados (closed → open → half-open), estado distribuido en la caché,
umbrales por circuito, excepciones ignoradas y eventos de ciclo de vida.

```bash
composer require webrek/laravel-circuit-breaker
```

## laravel-data-retention

Conserva los datos personales solo durante el tiempo que debes. Declara un
periodo de retención por modelo y qué ocurre cuando los registros lo superan —
eliminarlos o anonimizarlos — y un comando programado lo hace cumplir,
registrando cada fila que toca para tu rastro de cumplimiento.

```php
public function retentionPolicy(RetentionPolicy $policy): RetentionPolicy
{
    return $policy
        ->since('last_seen_at')->keepFor(365)            // un año de inactividad…
        ->where(fn ($q) => $q->where('legal_hold', false))
        ->anonymize([                                    // …luego limpia la PII
            'name'  => '[redacted]',
            'email' => fn ($c) => "anon+{$c->id}@example.test",
        ], markColumn: 'anonymized_at');
}
```

Acciones `delete()`, `forceDelete()` y `anonymize()`, scope de legal-hold, una
tabla de auditoría `data_retention_log` y comandos `retention:run` /
`retention:list`.

```bash
composer require webrek/laravel-data-retention
```

## laravel-mx-validation

Valida los identificadores mexicanos que recopilan tus formularios — RFC, CURP,
CLABE, NSS y código postal — con verificación real de dígito verificador, no
solo una regex.

```php
$request->validate([
    'rfc'   => ['required', 'rfc'],     // estructura + fecha + dígito verificador
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

Objetos de valor, casts de Eloquent y un proveedor de Faker para datos de
ejemplo válidos.

```bash
composer require webrek/laravel-mx-validation
```

## arco

Lleva el control de las **solicitudes ARCO** con su plazo legal y una **bitácora
de consentimiento** de solo-agregado (LFPDPPP). Núcleo independiente de framework
con un puente para Laravel.

```php
use Webrek\Arco\Laravel\Facades\{Arco, Consent};
use Webrek\Arco\ArcoRight;

$solicitud = Arco::receive(ArcoRight::Acceso, $user);   // registrada con su plazo
Consent::grant($user, 'marketing', ['privacy_version' => 'v3']);
```

```bash
composer require webrek/arco
```

## sat-69b

Verifica RFCs contra la **lista 69-B del SAT** (contribuyentes que facturan
operaciones simuladas — EFOS/EDOS). Núcleo independiente de framework con un
puente para Laravel que descarga y cachea el CSV oficial.

```php
use Webrek\Sat69b\Laravel\Facades\Sat69b;

Sat69b::isBlacklisted('AAA010101AA1');             // presunto o definitivo
Sat69b::check('AAA010101AA1')?->status->label();   // "Definitivo"
```

```bash
composer require webrek/sat-69b
```

## cfdi

Construye y **sella** CFDI 4.0 sobre [phpcfdi/eclipxe](https://github.com/phpcfdi),
y **timbra** detrás de un driver de PAC intercambiable. Núcleo independiente de
framework con un puente para Laravel.

```php
use Webrek\Cfdi\Laravel\Facades\Cfdi;
use Webrek\Cfdi\Csd;

$resultado = Cfdi::issue($creator, app(Csd::class));   // sella + timbra
$resultado->uuid;   // folio fiscal
```

```bash
composer require webrek/cfdi
```

---

## Usándolos juntos

Un solo flujo de pedido toca siete de ellos — seguro de reintentar, dinero
exacto, un ciclo de vida con guards, una funcionalidad gated, un pago protegido
por un circuit breaker, un evento publicado de forma confiable y salud
observable:

```php
// routes/web.php
Route::post('/orders', StoreOrderController::class)->middleware('idempotency');

// StoreOrderController
public function __invoke(Request $request)
{
    $total = Money::of($request->input('amount'), 'MXN');
    $total = $total->plus($total->percentage(16));                // money

    $order = DB::transaction(function () use ($request, $total) {
        $order = Order::create(['total' => $total]);              // estado inicializado en "pending"

        if (Features::active('instant-capture', $request->user())) {  // feature-flags
            CircuitBreaker::for('payments')->call(               // circuit-breaker
                fn () => $order->stateMachine()->apply('pay'),  // state-machine (atómico)
            );
        }

        Outbox::publish('order.placed', ['id' => $order->id]);   // outbox: hace commit junto con la orden
        return $order;
    });

    return response()->json($order, 201);                        // idempotency reproduce la respuesta al reintentar
}
```

## Calidad

Cada paquete viene con:

- Una suite de pruebas (PHPUnit + Orchestra Testbench)
- Análisis estático en **PHPStan nivel 6** (Larastan)
- Formateo con **Laravel Pint** (preset de Laravel)
- CI con GitHub Actions desde PHP 8.2 hasta 8.5 (Laravel 12 y 13)

## Licencia

Todos los paquetes se publican bajo la [licencia MIT](https://opensource.org/licenses/MIT).
