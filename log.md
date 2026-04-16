---
layout: page
title: Logging
permalink: /log/
---

# scafera/log

Structured logging for Scafera. PSR-3 implementation with a zero-dependency `StreamLogger` that writes JSON Lines to `var/log/{environment}.log`.

---

## Install

```bash
composer require scafera/log
```

---

## Core ideas

- **Application log calls are written by the developer at the call site** — no userland listeners, middleware, or automatic logging
- **Framework-level errors are captured automatically** — uncaught exceptions, console failures
- **`event` field convention** — every log call includes an `event` key, enforced by `scafera validate`
- **Fail loud I/O** — logging failures throw `\RuntimeException`. No silent degradation.

---

## Usage

Inject `Psr\Log\LoggerInterface` via constructor — it resolves to `StreamLogger` automatically:

```php
use Psr\Log\LoggerInterface;

final class OrderService
{
    public function __construct(
        private readonly LoggerInterface $logger,
    ) {}

    public function place(Order $order): void
    {
        // ... business logic ...

        $this->logger->info('Order placed', [
            'event' => 'order.created',
            'orderId' => $order->getId(),
        ]);
    }
}
```

---

## Event convention

Every log call should include an `event` key. Events use lowercase dot notation (`domain.action`):

```php
$this->logger->info('User signed in', ['event' => 'auth.login', 'userId' => 42]);
$this->logger->error('Payment failed', ['event' => 'payment.failed', 'orderId' => 1]);
$this->logger->warning('Slow query', ['event' => 'db.slow_query', 'duration_ms' => 1250]);
```

The `event` key is extracted from the context and promoted to a top-level JSON field. The remaining context is written under `context` (omitted if empty). This makes logs greppable and filterable via CLI commands.

**Reserved prefix:** `framework.*` — only the framework may emit these. Userland events must not use it.

---

## Log format

Each line is a self-contained JSON object:

```json
{"timestamp":"2026-04-10T16:22:40.501+00:00","level":"info","message":"Order placed","event":"order.created","ip":"192.168.1.42","context":{"orderId":1}}
```

Fields:

- `timestamp` — RFC3339 with milliseconds
- `level` — PSR-3 level
- `message` — message string
- `event` — present only when context includes an `event` key
- `ip` — client IP (HTTP requests only)
- `context` — remaining context after `event` extraction (omitted if empty)

### Exception serialization

`\Throwable` values under the `exception` key are expanded to structured data:

```php
$this->logger->error('Payment failed', [
    'event' => 'payment.failed',
    'exception' => $e,
]);
```

```json
{"timestamp":"...","level":"error","message":"Payment failed","event":"payment.failed","context":{"exception":{"class":"App\\Exception\\PaymentFailedException","message":"Card declined","code":0,"file":"/app/src/Service/PaymentService.php","line":42}}}
```

---

## Failure behavior

Logging failures throw `\RuntimeException`. Conscious trade-off: an unobservable success is worse than a visible failure (ADR-049).

**Storage implications:**

- Write to reliable local storage — local disk, tmpfs, or a volume backed by local block storage
- **NFS is not recommended** — network failure modes surface as `RuntimeException` and affect request handling
- **Rotation is delegated to the OS** — use `logrotate` (or equivalent). `StreamLogger` opens the log file fresh on each write, so post-rotation writes go to the new file without reload signals

Applications needing best-effort logging should use a different PSR-3 implementation.

---

## Framework error logging

When `scafera/log` is installed, uncaught exceptions are automatically logged alongside application entries. This is framework infrastructure — the log package takes ownership of error visibility.

**HTTP exceptions** — event `framework.http.error`:

- 4xx (client errors) logged at `warning` level
- 5xx and unhandled exceptions logged at `error` level

```json
{"timestamp":"...","level":"warning","message":"No route found for \"GET /nonexistent\"","event":"framework.http.error","context":{"exception":{...},"method":"GET","path":"/nonexistent","status":404}}
```

**Console exceptions** — event `framework.console.error` at `error` level, including command name and exit code.

Symfony's built-in error logging is disabled via a compiler pass to prevent duplicate entries. Symfony's error response handling (error page in dev, error controller in prod) continues to work normally.

---

## Build-time validation

`EventContextValidator` runs during `scafera validate`:

- Every logger call in `src/` must include `'event' =>` in the context
- Event values must match the format `domain.action` (lowercase dot notation: `^[a-z][a-z0-9]*(\.[a-z][a-z0-9]*)+$`)

The validator uses PHP's tokenizer. It handles single-line and multi-line logger calls, inspects only the top level of the context array, and format-checks event values only when they are string literals. Context built via a variable, method return, or array spread cannot be inspected and is silently skipped.

---

## CLI commands

```bash
vendor/bin/scafera logs:recent                        # latest 50 entries
vendor/bin/scafera logs:recent --limit=10             # latest 10
vendor/bin/scafera logs:recent --level=error          # filter by level
vendor/bin/scafera logs:recent --scope=app            # app logs only
vendor/bin/scafera logs:recent --scope=framework      # framework logs only

vendor/bin/scafera logs:status                        # 24h summary + top events + recent failures
vendor/bin/scafera logs:errors                        # severity >= error
vendor/bin/scafera logs:stats                         # counts grouped by event
vendor/bin/scafera logs:stats --by-level              # group by event + level separately

vendor/bin/scafera logs:filter order.created          # filter by event
vendor/bin/scafera logs:filter --level=warning
vendor/bin/scafera logs:filter --search="timeout"
vendor/bin/scafera logs:filter --level=error --scope=app    # AND logic

vendor/bin/scafera logs:clear                         # clear current log file
vendor/bin/scafera logs:clear -y                      # skip confirmation
```

All commands support `--json` for machine-readable output (meta + data envelope).

---

## Rules

- Inject `Psr\Log\LoggerInterface`, never `StreamLogger` directly
- Log in services, not controllers — controllers delegate to services
- Every log call is explicit and written by the developer — no automatic logging, no listeners, no middleware
- Do not catch or suppress logging exceptions — if I/O fails, the error must be visible

---

## Configuration

None. The bundle registers `StreamLogger` with `%kernel.logs_dir%` and `%kernel.environment%` — one logger, one file per environment.

Overriding the logger via `Psr\Log\LoggerInterface` alias is possible but **not recommended** — the CLI commands (`logs:stats`, `logs:filter`, `logs:errors`, `logs:status`) depend on the JSON Lines format that `StreamLogger` produces.
