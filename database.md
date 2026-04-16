---
layout: page
title: Database
permalink: /database/
---

# scafera/database

Persistence for Scafera. Wraps Doctrine ORM/DBAL internally — userland code never imports Doctrine types outside entities and repositories.

Capability package. Does not define folder structure — that belongs to architecture packages.

---

## Install

```bash
composer require scafera/database
```

The bundle is auto-discovered. Needs a `DATABASE_URL` — set it in `config/config.yaml` under `env:` or as an OS env var.

---

## Core API

Two classes cover 90% of cases:

- `EntityStore` — reads and writes
- `Transaction` — the only commit boundary

### EntityStore

```php
use Scafera\Database\EntityStore;

final class OrderRepository
{
    public function __construct(
        private readonly EntityStore $store,
    ) {}

    public function find(int $id): ?Order
    {
        return $this->store->find(Order::class, $id);
    }

    public function save(Order $order): void
    {
        $this->store->persist($order);
    }

    public function remove(Order $order): void
    {
        $this->store->remove($order);
    }
}
```

### Transaction

All writes go through `Transaction::run()`. Never call `flush()` manually. Unflushed writes are detected and throw at the end of the request.

```php
use Scafera\Database\EntityStore;
use Scafera\Database\Transaction;

final class OrderService
{
    public function __construct(
        private readonly EntityStore $store,
        private readonly Transaction $tx,
    ) {}

    public function place(Order $order): Order
    {
        return $this->tx->run(function () use ($order): Order {
            $this->store->persist($order);

            return $order;
        });
    }
}
```

Nested `Transaction::run()` calls are safe — inner calls use savepoints. Only the outermost level flushes and commits. If an inner call throws and an outer catches it, the inner changes roll back via savepoint while the outer transaction can still commit.

---

## Entities

Entities use `Scafera\Database\Mapping\Field\*` attributes. These map to Doctrine types internally but keep entities free of Doctrine imports.

```php
use Scafera\Database\Mapping\Auditable;
use Scafera\Database\Mapping\Field;

final class Order
{
    use Auditable;

    #[Field\Id]
    private ?int $id = null;

    public function __construct(
        #[Field\Varchar]
        private string $customerName,

        #[Field\Decimal(precision: 10, scale: 2)]
        private string $total,
    ) {
        $this->createdAt = new \DateTimeImmutable();
    }

    public function getId(): ?int
    {
        return $this->id;
    }
}
```

### Field attributes

`Id`, `Uuid`, `Varchar`, `VarcharShort`, `Text`, `Integer`, `IntegerBig`, `IntegerBigPositive`, `Boolean`, `Decimal`, `Money`, `Percentage`, `Date`, `DateTime`, `Time`, `UnixTimestamp`, `Json`.

Escape hatch for custom types:

```php
#[Field\Column(type: 'string', options: ['length' => 15])]
private string $isoCode;
```

### Auditable trait

Provides `$createdAt` and `$updatedAt`. You must initialize `$this->createdAt = new \DateTimeImmutable()` in the constructor — `AuditableInitValidator` warns if you forget.

### Table names

Derived as singular snake_case: `Order` → `order`, `BlogPost` → `blog_post`. To override:

```php
use Scafera\Database\Mapping\Table;

#[Table(name: 'categories')]
final class Category { /* ... */ }
```

---

## Repositories

Repositories are the **only zone** where direct Doctrine usage is allowed. `DoctrineBoundaryPass` enforces this at build time.

| Zone | Doctrine imports | Notes |
|------|-----------------|-------|
| `src/Entity/` | Forbidden | Use `Mapping\Field\*` attributes. Lifecycle callbacks rejected. |
| `src/Repository/` | Allowed | Controlled leakage — QueryBuilder, DQL, DBAL all permitted. |
| Everywhere else | Forbidden | Except `Doctrine\Common\Collections` (data structure, not behavioral). |

---

## Migrations

Migration files live in `support/migrations/` and extend `Scafera\Database\Migration`. Use the Schema API — no raw SQL, no Doctrine imports.

```php
use Scafera\Database\Migration;
use Scafera\Database\Schema\Schema;
use Scafera\Database\Schema\Table;

final class Version20260403080718 extends Migration
{
    public function up(Schema $schema): void
    {
        $schema->create('page', function (Table $table) {
            $table->id();
            $table->string('title', 255);
            $table->string('slug', 255);
            $table->text('content');
            $table->boolean('published');
            $table->timestamp('createdAt');
        });
    }

    public function down(Schema $schema): void
    {
        $schema->drop('page');
    }
}
```

### Column types

| Method | Doctrine type | Arguments |
|--------|--------------|-----------|
| `id()` | `integer` (auto-increment PK) | — |
| `string($name, $length)` | `string` | length (default 255) |
| `text($name)` | `text` | — |
| `integer($name)` | `integer` | — |
| `bigInteger($name)` | `bigint` | — |
| `smallInteger($name)` | `smallint` | — |
| `boolean($name)` | `boolean` | — |
| `timestamp($name)` | `datetime_immutable` | — |
| `date($name)` | `date_immutable` | — |
| `decimal($name, $precision, $scale)` | `decimal` | precision (default 8), scale (default 2) |
| `json($name)` | `json` | — |

### Modifiers

```php
$table->string('bio')->nullable();
$table->boolean('active')->default(true);
$table->string('notes')->nullable()->default('');
```

### Schema operations

```php
$schema->create('users', function (Table $table) { /* ... */ });
$schema->drop('users');                       // destructive
$schema->modify('users', function (Table $table) {
    $table->string('email', 255);             // add column
    $table->dropColumn('legacy_field');       // destructive
});
```

### Destructive detection

`db:migrate` classifies operations by type:

| Operation | Destructive |
|-----------|-------------|
| `CreateTable` | No |
| `AddColumn` | No |
| `DropTable` | Yes |
| `DropColumn` | Yes |

- **Development** (`APP_ENV=dev`): warns about destructive operations, proceeds
- **Production** (`APP_ENV=prod`): blocks execution unless `--force` is passed

Indexes and foreign key constraints are planned.

---

## Seeding

Seeders live in `support/seeds/` and implement `SeederInterface`. Class names must end with `Seed`:

```php
namespace App\Seed;

use Scafera\Database\EntityStore;
use Scafera\Database\SeederInterface;
use Scafera\Database\Transaction;

final class PageSeed implements SeederInterface
{
    public function __construct(
        private readonly EntityStore $store,
        private readonly Transaction $tx,
    ) {}

    public function run(): void
    {
        $this->tx->run(function (): void {
            $page = new \App\Entity\Page();
            $page->setTitle('Welcome');
            $page->setSlug('welcome');
            $this->store->persist($page);
        });
    }
}
```

Run seeders: `vendor/bin/scafera db:seed`.

---

## CLI commands

```bash
vendor/bin/scafera db:migrate                # Run pending migrations
vendor/bin/scafera db:migrate --dry-run      # Preview without running
vendor/bin/scafera db:migrate:create         # Blank migration file
vendor/bin/scafera db:migrate:diff           # Generate from entity/DB diff
vendor/bin/scafera db:migrate:drop <table>   # Generate a drop-table migration
vendor/bin/scafera db:migrate:status         # Status of applied / pending
vendor/bin/scafera db:migrate:rollback       # Rollback last migration
vendor/bin/scafera db:reset --force          # Drop all, re-run all (prod needs --force)
vendor/bin/scafera db:seed                   # Run seeders
vendor/bin/scafera db:schema:list            # List tables with column/row counts
vendor/bin/scafera db:schema:show <table>    # Show column definitions
vendor/bin/scafera db:schema:diff            # Show entity/DB mismatches
```

---

## Configuration

The bundle configures Doctrine automatically:

- **DBAL**: reads `DATABASE_URL` from env
- **ORM**: maps `App\Entity` from `src/Entity/` with attribute mapping
- **Migrations**: stored in `support/migrations/` under namespace `App\Migrations`

To override Doctrine config, add a `doctrine:` section to `config/config.yaml` (this leaks the engine name — a `database:` key is planned).

---

## Boundary rules

- No `Doctrine\*` imports in entities, services, controllers, or commands
- Repositories are the only zone where direct Doctrine usage is allowed
- Enforced at build time by `DoctrineBoundaryPass` — violations fail the container build

### Blocked imports

```
Scafera\Database\ScaferaDatabaseBundle
Scafera\Database\Migration\*                (except Scafera\Database\Migration itself)
Scafera\Database\Schema\*                   (except Scafera\Database\Schema\Schema)
Doctrine\*                                  (outside repositories)
```
