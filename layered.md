---
layout: page
title: Layered architecture
permalink: /layered/
---

# scafera/layered

Opinionated architecture package. Defines and enforces a strict, layered approach. All conventions are checked at validation time via `scafera validate` — violations are caught before code ships, not after.

All execution is explicit: no event subscribers, no listeners, no auto-discovered side effects. If behavior is not visible in the code being read, it does not happen.

---

## Architecture model

Six layers under `src/`:

| Layer | Purpose |
|-------|---------|
| `Controller/` | Single-action invokables — delegate to services, no business logic |
| `Service/` | All business logic |
| `Repository/` | Data access (Doctrine allowed — controlled zone) |
| `Integration/` | Third-party service wrappers (Stripe, Mailgun, external APIs) |
| `Entity/` | Domain data |
| `Command/` | CLI entry points via `#[AsCommand]` |

Plus two optional directories:

- `Form/` — Symfony Form types for complex forms (controlled zone, see [form](/form/))
- `Integration/` — only created when third-party integrations exist

---

## Dependency flow

Dependencies flow downward. Repository and Integration are siblings — both called by Service, neither depends on the other:

```
Controller → Service → Repository   → Entity
Command  ↗         ↘ Integration ↗
```

---

## Project structure

```
src/
├── Controller/    single-action, attribute routing
│   ├── Home.php
│   └── Order/
│       ├── Create.php
│       └── List.php
├── Command/       #[AsCommand], delegate to services
├── Service/       all business logic
├── Repository/    data access
├── Integration/   third-party wrappers (optional)
├── Form/          Symfony Form types (optional, controlled zone)
└── Entity/        domain data

tests/
├── Controller/    one test per controller, same path
├── Command/       one test per command, same path
└── Service/       unit tests where needed

resources/
└── templates/     Twig templates (when scafera/frontend installed)

support/
├── migrations/    database migrations
├── seeds/         database seeders
└── translations/  JSON translation files

config/
├── config.yaml
└── config.local.yaml   (gitignored)

var/
├── cache/
├── log/
└── uploads/
    ├── public/    files served to end users
    └── private/   files accessible only through app logic
```

Do not invent directories under `src/`. Do not put storage in the project root.

---

## Controllers

Controllers **must**:

- Be `final` classes
- Have a single `__invoke()` method — no other public methods
- Use action-oriented naming in domain subfolders: `Controller/Order/Show.php`, not `Controller/OrderController.php`
- Never use a `Controller` suffix
- Have a matching test: `src/Controller/Order/Show.php` → `tests/Controller/Order/ShowTest.php`
- Single-word names at root level only (e.g. `Controller/Home.php`) — multi-word names go in a subfolder

### Template

```php
<?php

declare(strict_types=1);

namespace App\Controller\Order;

use Scafera\Kernel\Http\JsonResponse;
use Scafera\Kernel\Http\Request;
use Scafera\Kernel\Http\Route;

#[Route('/orders/{id}', methods: 'GET', requirements: ['id' => '\d+'])]
final class Show
{
    public function __construct(
        private readonly OrderService $service,
    ) {}

    public function __invoke(Request $request): JsonResponse
    {
        return new JsonResponse($this->service->find((int) $request->routeParam('id')));
    }
}
```

### With Twig

```php
use Scafera\Kernel\Contract\ViewInterface;
use Scafera\Kernel\Http\Response;
use Scafera\Kernel\Http\Route;

#[Route('/', methods: 'GET')]
final class Home
{
    public function __construct(
        private readonly ViewInterface $view,
    ) {}

    public function __invoke(): Response
    {
        return new Response($this->view->render('home/index.html.twig', [
            'key' => 'value',
        ]));
    }
}
```

---

## Commands

Commands **must**:

- Be `final` classes
- Extend `Scafera\Kernel\Console\Command`
- Use `#[AsCommand('name', description: '...')]`
- Implement `handle(Input, Output): int` — return `self::SUCCESS` or `self::FAILURE`
- Have a matching test: `src/Command/SendWelcome.php` → `tests/Command/SendWelcomeTest.php`

### Template

```php
<?php

declare(strict_types=1);

namespace App\Command;

use Scafera\Kernel\Console\Attribute\AsCommand;
use Scafera\Kernel\Console\Command;
use Scafera\Kernel\Console\Input;
use Scafera\Kernel\Console\Output;

#[AsCommand('app:send-welcome', description: 'Send the welcome email')]
final class SendWelcome extends Command
{
    protected function handle(Input $input, Output $output): int
    {
        $output->success('Sent.');

        return self::SUCCESS;
    }
}
```

---

## Services

- Live in `src/Service/`
- Hold all business logic
- Must be `final`
- Use constructor injection for dependencies
- No HTTP or console concerns
- No framework imports

---

## Repositories

- Live in `src/Repository/`
- Must be `final`
- **Only zone** where direct Doctrine usage is allowed (QueryBuilder, DQL, DBAL)
- Used by services — never injected into controllers or commands directly

```php
use App\Entity\Order;
use Doctrine\ORM\EntityManagerInterface;

final class OrderRepository
{
    public function __construct(
        private readonly EntityManagerInterface $em,
    ) {}

    public function findByCustomer(string $name): array
    {
        return $this->em->getRepository(Order::class)
            ->findBy(['customerName' => $name]);
    }
}
```

---

## Integrations

- Live in `src/Integration/`, grouped by vendor (e.g. `Integration/Stripe/`)
- Wrap third-party SDKs and external APIs
- Must be `final`
- Class name must not repeat the vendor prefix (e.g. `Integration/Stripe/Payment.php` — **not** `StripePayment.php`)
- May depend on Entity and primitives only — no Service/Repository/Controller/Command imports
- Must not expose SDK types to callers — return domain objects or primitives
- Used by services — never injected into controllers or commands directly

See **[scafera/integration](/integration/)** for the Gateway pattern with HTTP-based third parties.

---

## Entities

- Live in `src/Entity/`
- Represent domain data only
- Use `Scafera\Database\Mapping\Field\*` attributes
- Use the `Auditable` trait for `createdAt`/`updatedAt` timestamps
- No `Doctrine\*` imports — enforced by `DoctrineBoundaryPass`

See **[scafera/database](/database/)** for field types and mapping details.

---

## Validators (enforced at build time)

Twelve validators run on every `scafera validate`:

| Validator | Rule |
|-----------|------|
| Tests directory | Tests must live in `tests/` only |
| Controller location | Controllers must live in `src/Controller/` |
| Controller naming | No `Controller` suffix; single-word at root, multi-word in subfolders |
| Single-action controllers | Must use `__invoke()`, no other public methods (except `__construct`) |
| Controller test parity | Every controller must have a matching test |
| Command test parity | Every command must have a matching test |
| Service location | Only recognized directories under `src/` |
| Service final | All services must be declared `final` |
| Namespace conventions | PSR-4 namespace must match file path |
| Layer dependencies | Enforces downward-only dependency flow |
| Integration naming | Vendor subfolder; class name must not repeat vendor prefix |
| No implicit execution | No `EventSubscriberInterface` or `#[AsEventListener]` in userland |

---

## Advisors (non-blocking hints)

| Advisor | What it checks |
|---------|---------------|
| Test sync | Warns when a controller or command is modified in git but its test is not |

The test-sync advisor requires git and skips gracefully with a reason when prerequisites are not met.

---

## Generators

```bash
vendor/bin/scafera make:controller Order/Create    # → src/Controller/Order/Create.php + test
vendor/bin/scafera make:service OrderService       # → src/Service/OrderService.php + test
vendor/bin/scafera make:command SendWelcome        # → src/Command/SendWelcome.php + test
```

Generators support nested names and reject convention-violating suffixes like `Controller` or `Command`.

---

## When to use / when to avoid

Use `scafera/layered` when you want strong conventions, predictable structure, and automated enforcement.

Avoid it if you need a flexible or custom architecture with minimal constraints.
