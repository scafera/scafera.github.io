---
layout: page
title: Translate
permalink: /translate/
---

# scafera/translate

UI string translation with locale management, RTL support, and Twig integration.

Internally adopts `symfony/translation`. Userland never imports Symfony Translation types — boundary enforcement blocks it at compile time.

---

## Install

```bash
composer require scafera/translate
```

---

## Core ideas

- **`{name}` parameter syntax, not `%name%`** — translation files use `{name}` for placeholders. Engine-independent, human-readable.
- **Translations directory is architecture-owned** — defined by `getTranslationsDir()` on the architecture package. For `scafera/layered`: `support/translations/`.
- **JSON only** — no YAML, no XLIFF. Flat key-value format.
- **Companion bundle for Twig** — `{{ t() }}` registered via `extra.scafera-bundles` when `scafera/frontend` is installed. The translate package works without Twig.
- **Missing keys return the raw key** — `get('MISSING')` returns `'MISSING'`. Fallback to default locale is automatic for keys that exist in another locale.

---

## Translation files

```json
{
    "WELCOME": "Welcome to the app!",
    "GREETING": "Hello, {name}!"
}
```

Location: `support/translations/{locale}.json`. Flat key-value format.

---

## Usage in services

```php
use Scafera\Translate\Translator;

$translator->get('WELCOME');                        // 'Welcome to the app!'
$translator->get('GREETING', ['name' => 'Alice']);  // 'Hello, Alice!'
$translator->has('WELCOME');                        // true
$translator->getLocale();                           // 'en'
$translator->getDirection();                        // 'ltr'
```

---

## Locale management

```php
use Scafera\Translate\LocaleManager;

$localeManager->setLocale('ar');
$localeManager->getLocale();              // 'ar'
$localeManager->getDirection();           // 'rtl'
$localeManager->getDirection('en');       // 'ltr'
$localeManager->getAvailableLocales();    // ['ar', 'en']
```

Available locales are discovered from `*.json` files in the translations directory.

---

## Twig integration

When `scafera/frontend` is installed, a companion bundle registers automatically:

```twig
<html dir="{{ locale_direction() }}">
    <h1>{{ t('WELCOME') }}</h1>
    <p>{{ t('GREETING', {name: user.name}) }}</p>
</html>
```

---

## Configuration

Optional. Defaults work out of the box.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `scafera.translate.default_locale` | `en` | Default locale. Also used as fallback for missing keys. |
| `scafera.translate.rtl_locales` | `[ar, fa, ur]` | Locales using right-to-left text direction. |

```yaml
# config/config.yaml
parameters:
    scafera.translate.default_locale: fr
    scafera.translate.rtl_locales: [ar, fa, ur]
```

**Not configurable:**

- Translation files directory — owned by the architecture package
- File format — JSON only
- Fallback chain — always falls back to the default locale (no multi-level fallback)

---

## Boundary enforcement

| Blocked | Use instead |
|---------|-------------|
| `Symfony\Component\Translation\*` | `Scafera\Translate\Translator` |

Enforced via compiler pass (build time) and validator (`scafera validate`).
