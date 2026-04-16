---
layout: page
title: Getting started
permalink: /getting-started/
---

# Getting started

Scafera is installed via Composer. The fastest way to start is the `scafera/layered-web` project template — it pulls in a working combination of architecture + capability + automation packages.

---

## Requirements

- PHP **8.4+**
- Composer 2
- A database engine (MySQL / MariaDB / PostgreSQL / SQLite) if you use `scafera/database`

Docker is recommended for local development. The project template ships with `Dockerfile` and `docker-compose.yml`.

---

## Create a project

```bash
composer create-project scafera/layered-web my-app
cd my-app
```

This bootstraps:

- `composer.json` with the full layered-web package set
- `public/index.php`, `bin/scafera`, kernel wiring — all scaffolded by `scafera/scaffold`
- `config/config.yaml` — the single project-level config file
- `Dockerfile` + `docker-compose.yml` (if using the Docker workflow)
- An `AGENTS.md` file — project-specific rules for AI agents

---

## With Docker

```bash
docker compose up -d --build
docker compose exec php composer install
docker compose exec php vendor/bin/scafera validate
docker compose exec php vendor/bin/phpunit -c tests/phpunit.dist.xml
```

Access the app at the port defined in `docker-compose.yml` (commonly `http://localhost:8080`).

---

## Without Docker

```bash
composer install
vendor/bin/scafera validate
vendor/bin/phpunit -c tests/phpunit.dist.xml
```

Start a dev server:

```bash
php -S 127.0.0.1:8080 -t public/
```

---

## Your first controller

Create `src/Controller/Hello.php`:

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use Scafera\Kernel\Http\JsonResponse;
use Scafera\Kernel\Http\Route;

#[Route('/hello', methods: 'GET')]
final class Hello
{
    public function __invoke(): JsonResponse
    {
        return new JsonResponse(['message' => 'Hello, Scafera']);
    }
}
```

Rules applied here (every controller, every time):

- `final` class — no inheritance
- Single `__invoke()` method — no other public methods
- No `Controller` suffix in the name
- Attribute routing via `#[Route]`
- No business logic — delegate to services when logic appears

---

## Matching test

Every controller requires a matching test at the same relative path. Create `tests/Controller/HelloTest.php`:

```php
<?php

declare(strict_types=1);

namespace App\Tests\Controller;

use Scafera\Kernel\Test\WebTestCase;

final class HelloTest extends WebTestCase
{
    public function testReturnsGreeting(): void
    {
        $response = $this->get('/hello');

        $response->assertOk();
        $response->assertJsonContains(['message' => 'Hello, Scafera']);
    }
}
```

Run it:

```bash
vendor/bin/phpunit -c tests/phpunit.dist.xml tests/Controller/HelloTest.php
```

Missing the test is a validator error — `scafera validate` will fail the build.

---

## Validate before committing

`scafera validate` is the single gate that runs every validator from every installed Scafera package. It catches boundary violations, structure violations, and missing tests:

```bash
vendor/bin/scafera validate
```

Run it in CI. Treat failures as build failures.

---

## Configuration

All project configuration lives in `config/config.yaml`. It has three sections:

```yaml
env:                          # per-environment (secrets, credentials, URLs)
    APP_SECRET: 's3cret-change-in-production'
    APP_DEBUG: '1'
    DATABASE_URL: 'mysql://user:pass@db:3306/app'

parameters:                   # app constants (don't vary per env)
    app.items_per_page: 25

framework:                    # bundle extension config
    default_locale: en
```

Real secrets go in `config/config.local.yaml` (gitignored). OS environment variables always win over both files.

**Do not** create `.env` files, `config/packages/`, or `config/bundles.php` — Scafera does not scan them.

See **[conventions](/conventions/)** for the full rule set.

---

## Generators

With `scafera/layered` installed, you can scaffold skeleton files:

```bash
vendor/bin/scafera make:controller Order/Create    # + matching test
vendor/bin/scafera make:service OrderService       # + matching test
vendor/bin/scafera make:command SendWelcome        # + matching test
```

Generators reject convention-violating names like `Controller` or `Command` suffixes.

---

## Next steps

- **[Conventions](/conventions/)** — the hard rules every Scafera project follows
- **[Layered architecture](/layered/)** — project structure and layer boundaries
- **[Database](/database/)** — entities, migrations, transactions
- **[Frontend](/frontend/)** — Twig via `ViewInterface`
