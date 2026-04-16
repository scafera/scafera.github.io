---
layout: page
title: Frontend
permalink: /frontend/
---

# scafera/frontend

Template rendering for Scafera. Wraps Twig internally — your code never touches Twig directly.

Capability package. Provides `ViewInterface` implementation.

---

## Install

```bash
composer require scafera/frontend
```

Most projects pull it in via `scafera/layered-web`.

---

## Contract

The kernel defines the interface — the only type your code depends on:

```php
namespace Scafera\Kernel\Contract;

interface ViewInterface
{
    public function render(string $template, array $context = []): string;
}
```

`render()` returns a **string**, not a Response. The controller wraps it — this lets views be reused for emails, PDFs, etc.

---

## Usage in controllers

Inject `ViewInterface` via constructor:

```php
use Scafera\Kernel\Contract\ViewInterface;
use Scafera\Kernel\Http\Request;
use Scafera\Kernel\Http\Response;
use Scafera\Kernel\Http\Route;

#[Route('/orders/{id}', methods: 'GET')]
final class Show
{
    public function __construct(
        private readonly ViewInterface $view,
        private readonly OrderService $orders,
    ) {}

    public function __invoke(Request $request): Response
    {
        $order = $this->orders->find($request->routeParam('id'));

        return new Response($this->view->render('order/show.html.twig', [
            'order' => $order,
        ]));
    }
}
```

---

## Templates

- Live in `resources/templates/` at the project root
- Use Twig's native {% raw %}`{% extends %}`, `{% block %}`, `{% include %}`{% endraw %} — no custom helpers

### Layout

{% raw %}
```twig
{# resources/templates/base.html.twig #}
<!doctype html>
<html>
<head>
    <title>{% block title %}Scafera{% endblock %}</title>
</head>
<body>
    {% block body %}{% endblock %}
</body>
</html>
```

### Page

```twig
{# resources/templates/order/show.html.twig #}
{% extends 'base.html.twig' %}

{% block title %}Order #{{ order.id }}{% endblock %}

{% block body %}
    <h1>Order #{{ order.id }}</h1>
    <p>{{ order.customerName }}</p>
{% endblock %}
```
{% endraw %}

---

## Boundary enforcement

`TwigLeakageValidator` scans `src/` for direct `Twig\*` imports. Violations surface in `scafera validate`:

```
Package checks:
  ✗ No Twig imports in userland FAILED
    - src/Service/PdfGenerator.php: imports Twig types directly — use Scafera\Kernel\Contract\ViewInterface instead
```

Inject `ViewInterface`. Do not import `Twig\Environment`, `Twig\Loader\*`, or any other Twig class.

---

## Twig functions from other packages

When installed together, capability packages register Twig functions via companion bundles:

| From | Function | Purpose |
|------|----------|---------|
| `scafera/asset` | `asset('path')` | URL for a file under `assets/` |
| `scafera/translate` | `t('KEY', {param: value})` | Translate |
| `scafera/translate` | `locale_direction()` | `'ltr'` or `'rtl'` |

See the respective package pages for details.

---

## What this package does NOT own

- **Form rendering** — `scafera/form` (plain HTML; no form themes)
- **Asset management** — `scafera/asset`
- **Twig extensions for userland** — not supported (ADR-030)
- **View composers / shared template data** — inject services explicitly
- **Layout helpers** — use Twig's native {% raw %}`{% extends %}` and `{% block %}`{% endraw %}
