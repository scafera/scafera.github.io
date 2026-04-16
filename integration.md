---
layout: page
title: Integration
permalink: /integration/
---

# scafera/integration

External system communication for Scafera. An enforceable **gateway pattern** for calling third-party APIs — all behind Scafera-owned types.

Internally adopts `symfony/http-client`. Userland never imports Symfony HttpClient types — boundary enforcement blocks it at build time. All alternative HTTP mechanisms (cURL, `file_get_contents` with HTTP URLs, `fopen` with HTTP URLs) are also blocked.

---

## Install

```bash
composer require scafera/integration
```

---

## Core idea

Your application code interacts with **gateways** — classes with business-level methods like `createPayment()` or `getRate()` — never making raw HTTP calls. Each gateway wraps one external system, receives a configured `HttpClient` via the `#[Integration]` attribute, and uses relative paths against a pre-configured base URL.

A build-time validator blocks Symfony HttpClient, cURL, and HTTP-targeted `file_get_contents`/`fopen` everywhere in `src/`. The `HttpClient` class itself is only allowed inside the `Integration/` layer.

---

## Configuration

```yaml
# config/config.yaml
integration:
    stripe:
        base_url: 'https://api.stripe.com/v1'
        auth: ''
    mailgun:
        base_url: 'https://api.mailgun.net/v3'
        auth: ''
```

```yaml
# config/config.local.yaml (gitignored)
integration:
    stripe:
        auth: 'Bearer sk_live_real_secret'
    mailgun:
        auth: 'Basic key-real_secret'
```

If an integration has different base URLs per environment (sandbox vs production), override `base_url` in `config.local.yaml` as well.

---

## Gateway

One class per external system with business-level methods:

```php
namespace App\Integration\Stripe;

use Scafera\Integration\HttpClient;
use Scafera\Integration\Attribute\Integration;

final class PaymentGateway
{
    public function __construct(
        #[Integration('stripe')]
        private HttpClient $http,
    ) {}

    public function createPayment(int $amount, string $currency): array
    {
        return $this->http->post('/charges', [
            'amount' => $amount,
            'currency' => $currency,
        ])->json();
    }

    public function refund(string $chargeId): array
    {
        return $this->http->post('/refunds', [
            'charge' => $chargeId,
        ])->json();
    }
}
```

### Gateway rules

- Class name must end with `Gateway` — enforced by validator
- One class per external system
- Business-level methods only — `createPayment()`, not `post()`
- No HTTP types in public method signatures — return arrays or domain objects
- No full URLs — endpoint paths only, base URL from config

---

## Using a gateway in a service

```php
namespace App\Service\Order;

use App\Integration\Stripe\PaymentGateway;

final class PlaceOrder
{
    public function __construct(
        private PaymentGateway $payment,
    ) {}

    public function handle(int $amount): array
    {
        return $this->payment->createPayment($amount, 'usd');
    }
}
```

---

## Integration-specific config values

Integrations can carry arbitrary config values beyond `base_url` and `auth`. They are injected via the same `#[Integration]` attribute with a second argument:

```yaml
# config/config.yaml
integration:
    linkedin:
        base_url: 'https://api.linkedin.com/v2'
        auth: ''
        contract_id: ''
        seat_limit: 500
```

```php
namespace App\Integration\LinkedIn;

use Scafera\Integration\HttpClient;
use Scafera\Integration\Attribute\Integration;

final class PremiumGateway
{
    public function __construct(
        #[Integration('linkedin')]
        private HttpClient $http,

        #[Integration('linkedin', 'contract_id')]
        private string $contractId,

        #[Integration('linkedin', 'seat_limit')]
        private int $seatLimit,
    ) {}
}
```

- `#[Integration('name')]` — resolves to the `HttpClient` service
- `#[Integration('name', 'key')]` — resolves to a config value

---

## `HttpClient` API

```php
$this->http->get(string $path, array $options = []): Response;
$this->http->post(string $path, array $data = [], array $options = []): Response;
$this->http->put(string $path, array $data = [], array $options = []): Response;
$this->http->patch(string $path, array $data = [], array $options = []): Response;
$this->http->delete(string $path, array $options = []): Response;
```

`$data` is sent as JSON body. For other content types (form-encoded, multipart), use the `$options` array directly, e.g. `$this->http->post('/upload', [], ['body' => $formData])`.

---

## `Response` API

```php
$response->statusCode(): int;
$response->json(): array;
$response->body(): string;
$response->headers(): array;
$response->header(string $name): ?string;
```

HTTP errors do **not** throw automatically — error handling is the gateway's responsibility.

---

## Testing

### Services that use gateways

Mock the gateway, not the HTTP client. Stable, behavior-focused — no HTTP client mocking, no request/response faking.

### Gateways themselves

The `HttpClient` constructor accepts an optional `HttpClientInterface` for testing — the bundle never passes it in production (engine stays hidden):

```php
use Symfony\Component\HttpClient\MockHttpClient;
use Symfony\Component\HttpClient\Response\MockResponse;

$mock = new MockHttpClient([new MockResponse('{"id": 1}')]);
$http = new HttpClient('https://api.example.com', null, $mock);

$gateway = new PaymentGateway($http);
$result = $gateway->createPayment(1000, 'usd');
```

---

## Boundary enforcement

| Blocked | Use instead |
|---------|-------------|
| `Symfony\Contracts\HttpClient\*` | `HttpClient` in a Gateway class |
| `Symfony\Component\HttpClient\*` | `HttpClient` in a Gateway class |
| `curl_*` functions | `HttpClient` in a Gateway class |
| `file_get_contents` with HTTP URLs | `HttpClient` in a Gateway class |
| `fopen` with HTTP URLs | `HttpClient` in a Gateway class |
| `Scafera\Integration\HttpClient` outside `Integration/` | Inject the gateway, not the HTTP client |
| `#[Integration('name', 'key')]` outside `Integration/` | Integration config values belong in gateways only |
| Unused config keys under `integration:` | Remove unused keys or reference them in a gateway |

Enforced via validators (`scafera validate`).
