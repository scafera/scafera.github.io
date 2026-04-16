---
layout: page
title: Scaffold
permalink: /scaffold/
---

# scafera/scaffold

Composer plugin that scaffolds framework-owned files into Scafera projects.

> **This is not a code generator.** It does not create controllers, services, or commands — for that, use `vendor/bin/scafera make:controller`, `make:service`, `make:command` (provided by architecture packages like `scafera/layered`).
>
> This plugin is a Composer hook that copies framework-owned files (entry point, kernel bootstrap, `.gitignore`, config examples) into your project on `composer install` so the framework and your project stay in sync.

Runs automatically on `composer install` and `composer update`. Scaffolded files are always overwritten to prevent drift between framework and project.

---

## How it works

Scafera packages declare which files they provide. The plugin collects these declarations from all installed packages and copies the files into the project.

```
composer install
  → plugin reads declarations from installed packages
  → copies files to project
  → creates directories as needed
```

---

## Package declarations

Packages declare scaffold files in their `composer.json` under `extra.scafera-scaffold`.

### `files` — always overwritten

Use a logical key (not a path) to identify each file. The key decouples the file identity from its target location.

```json
{
    "extra": {
        "scafera-scaffold": {
            "files": {
                "index.php": "support/scaffold/public/index.php"
            }
        }
    }
}
```

The logical key is the default target path. To place the file elsewhere, an architecture package provides a `target-map`.

### `initial-files` — created once

Only copied if the target does not already exist. Useful for configuration files the developer is expected to modify.

```json
{
    "extra": {
        "scafera-scaffold": {
            "initial-files": {
                "config/config.yaml": "support/scaffold/config/config.example.yaml"
            }
        }
    }
}
```

Unlike `files`, `initial-files` use literal target paths (not logical keys).

### `target-map` — control placement

Architecture packages can remap where a logical key is placed without duplicating the source file.

```json
{
    "extra": {
        "scafera-scaffold": {
            "target-map": {
                "index.php": "public/index.php"
            }
        }
    }
}
```

The source remains in the kernel — no duplication. A different architecture package could remap the same key:

```json
{
    "target-map": {
        "index.php": "web/index.php"
    }
}
```

---

## Project-level overrides

Projects can disable specific scaffolded files via `file-mapping`:

```json
{
    "extra": {
        "scafera-scaffold": {
            "file-mapping": {
                "public/index.php": false
            }
        }
    }
}
```

Disabled files are skipped with a console message.

---

## File placement convention

Packages that provide scaffold files place them under `support/scaffold/` at the package root:

```
my-package/
├── src/
├── support/
│   └── scaffold/
│       └── public/
│           └── index.php
└── composer.json
```

---

## Conflict resolution

If multiple packages declare the same logical key, the last package processed wins. Package order is determined by Composer's dependency resolution.

---

## Console output

```
Scafera: scaffolding project files...
  .gitignore → .gitignore (from scafera/layered)
  config/config.example.yaml → config/config.example.yaml (from scafera/kernel)
  index.php → public/index.php (from scafera/kernel)
  Skipped: .gitignore (already exists)
Scafera: 3 file(s) scaffolded.
```
