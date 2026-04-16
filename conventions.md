---
layout: page
title: Conventions
permalink: /conventions/
---

# Conventions

These rules apply to **every Scafera project**, regardless of which architecture or capability packages are installed. They are enforced by validators and compiler passes — `scafera validate` surfaces violations as build failures.

---

## Hard rules

- `declare(strict_types=1);` in every PHP file
- **Never create:**
  - `src/Kernel.php` — owned by the framework
  - `public/index.php` — scaffolded by `scafera/scaffold`
  - `bin/console` — use `vendor/bin/scafera`
  - `config/packages/` — the kernel does not scan it
  - `config/bundles.php` — bundles are auto-discovered from `installed.json`
  - `.env` files — Scafera does not use dotenv
- **Never create** middleware, event listeners, subscribers, or use `#[AsEventListener]` in userland — all execution must be explicit
- **Never import framework engines** (`Symfony\…`, `Doctrine\…`, `Twig\…`) in userland — use the Scafera API only
- Use constructor injection — no service locators, no manual wiring
- Controllers are stateless — no static or shared mutable state
- Adding a controller or command without a matching test is a violation

---

## Boundary enforcement

Each capability package blocks the engine it wraps:

| Package | Blocked in userland | Use instead |
|---------|--------------------|-------------|
| `scafera/database` | `Doctrine\*` (except in `src/Repository/`) | `EntityStore`, `Transaction`, `Mapping\Field\*` |
| `scafera/frontend` | `Twig\*` | `ViewInterface::render()` |
| `scafera/auth` | `Symfony\…\Security\*`, `…\Session\*`, `…\PasswordHasher\*` | `Authenticator`, `Session`, `Password`, `#[Protect]` |
| `scafera/form` | `Symfony\…\Form\*`, `…\Validator\*` (except in `src/Form/`) | DTO + `Rule\*` + `FormHandler` |
| `scafera/file` | `Symfony\…\File\*`, `…\Filesystem\*` | `UploadExtractor`, `FileStorage`, `FileResponse` |
| `scafera/translate` | `Symfony\…\Translation\*` | `Translator`, `LocaleManager` |
| `scafera/asset` | `Symfony\…\AssetMapper\*` | `asset()` in Twig |
| `scafera/integration` | `Symfony\…\HttpClient\*`, `curl_*`, HTTP in `file_get_contents`/`fopen` | Gateway + `HttpClient` |

Boundary violations are **compile-time** failures — the container does not build.

---

## Controlled zones

Two directories permit engine imports, by exception:

- `src/Repository/` — direct Doctrine usage allowed (QueryBuilder, DQL, DBAL)
- `src/Form/` — Symfony Form types allowed for forms beyond DTO capability

These are **not escape hatches**. They are documented zones with their own rules:

- Only controllers may import from `App\Form`
- Repositories are called by services — never injected into controllers or commands

---

## Allowed imports (Scafera API)

Full listing lives on each package page. Quick reference:

```
HTTP        Scafera\Kernel\Http\{Request,Response,JsonResponse,RedirectResponse,Route}
View        Scafera\Kernel\Contract\ViewInterface
Console     Scafera\Kernel\Console\{Command,Input,Output}
            Scafera\Kernel\Console\Attribute\AsCommand
DI          Scafera\Kernel\Attribute\Config
Testing     Scafera\Kernel\Test\{WebTestCase,TestResponse,CommandTestCase,CommandResult}

Database    Scafera\Database\{EntityStore,Transaction,Migration,SeederInterface}
            Scafera\Database\Mapping\{Table,Auditable,Field\*}
            Scafera\Database\Schema\Schema

Auth        Scafera\Auth\{Session,Authenticator,Password,CookieJar,Protect}
            Scafera\Auth\{SessionGuard,RoleGuard,GuardInterface,User,UserProvider}

Form        Scafera\Form\{FormHandler,Form,Validator,ValidationResult,ValidationError}
            Scafera\Form\Rule\*

File        Scafera\File\{UploadExtractor,UploadedFile,UploadConstraint}
            Scafera\File\{UploadValidator,UploadResult,FileStorage,FileResponse}

Translate   Scafera\Translate\{Translator,LocaleManager}

Integration Scafera\Integration\HttpClient            (only inside src/Integration/)
            Scafera\Integration\Attribute\Integration

Logging     Psr\Log\LoggerInterface
```

**Do not import** `*\*Bundle`, `*\Listener\*`, `*\Internal\*` from any Scafera package.

---

## Configuration

Single file: `config/config.yaml`.

```yaml
env:
    APP_SECRET: 's3cret-change-in-production'
    APP_DEBUG: '1'
    DATABASE_URL: 'mysql://user:pass@db:3306/app'

parameters:
    app.items_per_page: 25

framework:
    default_locale: en
```

- **`env:`** — varies per environment (secrets, URLs, credentials). OS env vars override this.
- **`parameters:`** — application constants that don't vary per environment.
- **Bundle extensions** (`framework:`, `doctrine:`, etc.) — the remaining top-level keys.
- **`config.local.yaml`** — gitignored overrides for secrets and per-machine settings.

Inject config into services:

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

---

## Paths

Framework-wide paths are fixed. Do not invent alternatives:

| Path | Purpose | Owner |
|------|---------|-------|
| `public/` | HTTP document root | kernel |
| `config/` | `config.yaml`, `config.local.yaml` | kernel |
| `var/cache/`, `var/log/` | Runtime state | kernel |
| `support/migrations/` | Database migrations (`App\Migrations`) | database |
| `support/seeds/` | Database seeders (`App\Seed`) | database |
| `support/scaffold/` | Files provided by scaffold plugin | scaffold |

Architecture packages own `src/` and `resources/` layout. For layered: `src/{Controller,Service,Repository,Integration,Entity,Command,Form}/`, templates in `resources/templates/`, translations in `support/translations/`, assets in `assets/`.

**Never hardcode framework-defined paths.** Use `vendor/bin/scafera info:paths` to discover them at runtime.

---

## Agent discipline

These rules govern how you work, not what you build.

- Only change what the task requires — no refactoring, renaming, or "improvements" in untouched code
- Do not add comments, docstrings, or type annotations to code you didn't change
- Keep diffs minimal — every change must have a clear functional purpose
- Use only the directories defined by the architecture package — do not invent new ones
- Prefer explicit code over magic — behavior should be visible at the call site
- If something seems beneficial but wasn't requested, **ask first** — do not add silently
- When unsure about paths or structure, run `vendor/bin/scafera info:paths` and `vendor/bin/scafera validate` — do not guess

---

## Running `scafera` commands

All projects ship with `vendor/bin/scafera` (from `scafera/kernel`):

```bash
vendor/bin/scafera validate                # run all validators/advisors
vendor/bin/scafera info:paths              # show all registered paths
vendor/bin/scafera info:paths storage      # show a single path
vendor/bin/scafera about                   # framework + environment info
```

Package commands are namespaced:

```bash
vendor/bin/scafera db:migrate              # scafera/database
vendor/bin/scafera logs:recent             # scafera/log
vendor/bin/scafera make:controller Foo     # scafera/layered
```

With Docker, prefix with `docker compose exec php`.

---

## Final rule

> If a change weakens boundaries, increases complexity, or introduces ambiguity — reject it.

Keep Scafera projects **small**, **strict**, and **consistent**.
