---
layout: page
title: Kernel
permalink: /kernel/
---

# scafera/kernel

> The boot core of Scafera. Discovers bundles, loads an architecture package, enforces structural boundaries, and hands off to Symfony. **User projects never define a `Kernel` class.**

The kernel is intentionally non-functional without an architecture package (e.g. `scafera/layered`). Without one: no user services registered, no routes loaded, only built-in commands (`about`, `validate`, `cache:clear`) work.

---

## What it provides

- `ScaferaKernel` — the single boot class
- `vendor/bin/scafera` — CLI entry point
- `Bootstrap` — pre-boot environment preparation
- `ArchitecturePackageInterface` — the extension point that turns the headless kernel into a working app

---

## How it works

### Dynamic bundle discovery

Bundles are discovered automatically from Composer's `installed.json`. Any installed package declaring `"type": "symfony-bundle"` is registered at boot — no `config/bundles.php` needed.

- Install a capability package and its bundle is available
- Remove the package and the bundle disappears
- `composer install --no-dev` excludes dev bundles naturally

### Two-phase config parsing

`config/config.yaml` is parsed twice:

1. **Bootstrap phase** — reads `env:` before the kernel boots (line parser, no YAML library). Sets values as `$_ENV`/`$_SERVER`. OS env vars take precedence.
2. **Container build phase** — re-parses with Symfony's YAML parser. Strips `env:`, processes `parameters:` into the container, passes remaining keys (e.g. `framework:`, `doctrine:`) as bundle extension configs.

Only valid bundle extension names are allowed alongside `env:` and `parameters:`. Unknown keys cause an error.

### `APP_SECRET`

`APP_SECRET` has no default. It must be set via `config/config.yaml` under `env:` or as an OS environment variable. `Bootstrap` throws `RuntimeException` if missing.

---

## HTTP types

Controllers use these types instead of Symfony's HTTP classes. `ControllerBoundaryPass` enforces this at compile time.

| Type | Purpose |
|------|---------|
| `Scafera\Kernel\Http\Request` | Incoming HTTP request |
| `Scafera\Kernel\Http\Response` | Plain response |
| `Scafera\Kernel\Http\JsonResponse` | JSON response |
| `Scafera\Kernel\Http\RedirectResponse` | Redirect response |
| `Scafera\Kernel\Http\Route` | `#[Route]` attribute |
| `Scafera\Kernel\Http\ParameterBag` | Query/request params (typed getters) |
| `Scafera\Kernel\Http\HeaderBag` | Request headers (case-insensitive) |

### Request

Properties:
- `$request->query` — `ParameterBag` of query string parameters
- `$request->request` — `ParameterBag` of POST body parameters
- `$request->headers` — `HeaderBag` of request headers

Methods:
- `getMethod(): string`
- `isMethod(string $method): bool`
- `getPath(): string` — URL path without query string
- `getUri(): string`
- `getContent(): string` — raw body
- `json(): array` — parsed JSON body
- `routeParam(string $key, mixed $default = null): mixed`
- `routeParams(): array`

### Response types

```php
new Response(string $content = '', int $status = 200, array $headers = []);
new JsonResponse(mixed $data = null, int $status = 200, array $headers = []);
new RedirectResponse(string $url, int $status = 302, array $headers = []);
```

### `#[Route]`

```php
#[Route(
    path: '/path/{param}',
    methods: 'GET',                      // string or array
    name: 'custom_name',                 // optional, auto-generated if omitted
    requirements: ['param' => '\d+'],
    defaults: ['param' => '1'],
)]
```

Class-level `#[Route]` sets a prefix for method-level routes. A class-level `#[Route]` with no method-level routes maps to `__invoke`.

### ParameterBag

- `get(string $key, mixed $default = null): mixed`
- `has(string $key): bool`
- `all(): array`
- `getInt`, `getString`, `getBoolean` — typed getters

### HeaderBag

- `get(string $key, ?string $default = null): ?string` — case-insensitive
- `has(string $key): bool`
- `all(): array`

---

## Console

| Type | Purpose |
|------|---------|
| `Scafera\Kernel\Console\Command` | Base command — implement `handle()` |
| `Scafera\Kernel\Console\Input` | Input wrapper |
| `Scafera\Kernel\Console\Output` | Output wrapper — `success()`, `error()`, `warning()`, `info()`, `table()`, `note()` |
| `Scafera\Kernel\Console\Attribute\AsCommand` | `#[AsCommand('name', description: '...')]` |

Command provides: `addArg($name, $description, $required, $default)`, `addFlag($name, $shortcut, $description)`, `addOpt($name, $shortcut, $description, $default)`.

Implement `handle(Input $input, Output $output): int` — return `self::SUCCESS` or `self::FAILURE`.

---

## Testing

| Type | Purpose |
|------|---------|
| `Scafera\Kernel\Test\WebTestCase` | HTTP test base — `get()`, `post()`, `postJson()`, `put()`, `putJson()`, `delete()`, `patch()`, `patchJson()` |
| `Scafera\Kernel\Test\TestResponse` | Fluent assertions — `assertOk()`, `assertStatus()`, `assertSuccessful()`, `assertRedirect()`, `assertHeader()`, `assertHeaderExists()`, `assertJsonContains()`, `assertJsonPath()`, `assertContentContains()`, `getContent()`, `getStatusCode()`, `json()` |
| `Scafera\Kernel\Test\CommandTestCase` | Console test base — `runCommand('name', ['arg' => 'value'])` |
| `Scafera\Kernel\Test\CommandResult` | Command assertions — `assertSuccessful()`, `assertFailed()`, `assertExitCode()`, `assertOutputContains()`, `assertOutputNotContains()`, `getOutput()`, `getExitCode()` |

---

## Dependency injection

Inject configuration with `#[Config]`:

```php
use Scafera\Kernel\Attribute\Config;

final class SomeService
{
    public function __construct(
        #[Config('app.items_per_page')] private readonly int $perPage,
        #[Config('env.DATABASE_URL')] private readonly string $dbUrl,
    ) {}
}
```

- `#[Config('app.name')]` — container parameter
- `#[Config('env.NAME')]` — environment variable

---

## Contracts

The kernel defines interfaces that architecture and capability packages implement:

| Contract | Purpose |
|----------|---------|
| `ArchitecturePackageInterface` | Defines an architecture package |
| `ValidatorInterface` | Hard validation — affects exit code |
| `AdvisorInterface` | Soft advisory check — never affects exit code |
| `GeneratorInterface` | Code generator for `scafera make:*` commands |
| `ViewInterface` | Template rendering (implemented by `scafera/frontend`) |
| `PathProviderInterface` | Contributes paths to `info:paths` |

DI tags auto-collected: `scafera.validator`, `scafera.advisor`, `scafera.path_provider`.

---

## What the kernel does NOT own

- **Folder conventions** — architecture packages
- **Presentation** — `scafera/frontend`
- **Persistence** — `scafera/database`
- **Logging** — `scafera/log`
- **HTTP header / CORS customization** — dedicated bundles via `config.yaml`
- **Business logic** — your project
