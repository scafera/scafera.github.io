---
layout: page
title: Assets
permalink: /asset/
---

# scafera/asset

Asset management for Scafera. Configures Symfony's AssetMapper internally — your code references assets via `asset()` in Twig templates, never through PHP imports.

Capability package (adoption gate). It activates AssetMapper with sensible defaults, provides boundary enforcement, and declares companion bundles for the asset ecosystem — so tools like TailwindBundle work out of the box with zero configuration.

---

## Install

```bash
composer require scafera/asset
```

Requires `scafera/frontend` (for Twig `asset()` function).

---

## Reference assets in templates

Place CSS, JS, and other static files in `assets/` at the project root. Reference them in Twig:

{% raw %}
```twig
<link rel="stylesheet" href="{{ asset('styles/app.css') }}">
<script src="{{ asset('js/app.js') }}"></script>
<img src="{{ asset('images/logo.png') }}">
```
{% endraw %}

---

## With Tailwind CSS

Install TailwindBundle — it is auto-registered as a companion bundle:

```bash
composer require symfonycasts/tailwind-bundle
```

Initialize and build:

```bash
vendor/bin/scafera symfony tailwind:init
vendor/bin/scafera symfony tailwind:build
```

Watch for changes during development:

```bash
vendor/bin/scafera symfony tailwind:build --watch
```

Reference the compiled CSS:

{% raw %}
```twig
<link rel="stylesheet" href="{{ asset('styles/app.css') }}">
```
{% endraw %}

---

## Production build

Compile assets with versioned filenames for cache busting:

```bash
vendor/bin/scafera symfony asset-map:compile
```

Versioned files are written to `public/assets/` for direct serving by the web server.

---

## Companion bundles

This package declares companion bundles via `extra.scafera-bundles` in its `composer.json`. When you install a companion package, Scafera registers its bundle automatically — no manual configuration needed.

| Package | Bundle | Purpose |
|---------|--------|---------|
| `symfonycasts/tailwind-bundle` | `SymfonycastsTailwindBundle` | Tailwind CSS compilation via standalone binary |

Companions are only registered when installed. If you don't `composer require` the package, the declaration is ignored.

---

## Configuration

The bundle configures AssetMapper automatically:

- **Asset paths**: `assets/` at your project root
- **Public prefix**: `/assets/`

To override defaults:

```yaml
# config/config.yaml
framework:
    asset_mapper:
        paths:
            - 'assets/'
            - 'vendor/some-package/assets/'
```

---

## Boundary enforcement

`AssetMapperLeakageValidator` scans `src/` for direct `Symfony\Component\AssetMapper\*` imports. Violations surface in `scafera validate`:

```
Package checks:
  ✗ No AssetMapper imports in userland FAILED
    - src/Service/AssetHelper.php: imports AssetMapper types directly — use asset() in Twig templates instead
```

Use `asset()` in templates. Do not import AssetMapper types in PHP.

---

## What this package does NOT own

- **Template rendering** — `scafera/frontend`
- **JavaScript bundling** — AssetMapper uses native ES modules, no bundler needed
- **Node.js tooling** — TailwindBundle downloads its own standalone binary
