---
layout: home
title: Scafera
---

# Scafera

> A predictable, enforceable PHP framework designed so humans **and AI agents** can build apps in it without breaking its rules.

Scafera wraps Symfony — the kernel, entry points, and bundle discovery are owned by the framework. Your code contains only business logic. Rules are **enforced by the system** (validators, compiler passes, `scafera validate`), not suggested by documentation.

Why this matters for AI-assisted development: frameworks with "conventions" depend on the developer to follow them. Agents drift. Scafera makes the rules build-time failures — the agent either produces correct code or the build fails with a clear error.

---

## Start here

- **[Getting started](getting-started.html)** — bootstrap a project from `scafera/layered-web`
- **[Conventions](conventions.html)** — the hard rules every project follows
- **[Layered architecture](layered.html)** — project structure, layers, validators
- **[Kernel](kernel.html)** — HTTP, Console, Testing, contracts

---

## Package catalog

### Foundation

| Package | Purpose |
|---------|---------|
| [`scafera/kernel`](kernel.html) | Core runtime — HTTP, Console, Testing, contracts |

### Architecture

| Package | Purpose |
|---------|---------|
| [`scafera/layered`](layered.html) | Layered architecture — Controller / Service / Repository / Integration / Entity / Command |

### Capabilities

| Package | Purpose |
|---------|---------|
| [`scafera/database`](database.html) | Persistence — EntityStore, Transaction, Migrations (Doctrine internal) |
| [`scafera/frontend`](frontend.html) | View rendering — `ViewInterface` (Twig internal) |
| [`scafera/auth`](auth.html) | Authentication — Session, Authenticator, Guards |
| [`scafera/form`](form.html) | Forms — DTO validation, FormHandler, CSRF |
| [`scafera/file`](file.html) | Files — Upload, Validate, Store, Serve |
| [`scafera/translate`](translate.html) | i18n — JSON files, locale, RTL, Twig functions |
| [`scafera/asset`](asset.html) | Assets — AssetMapper + `asset()` in Twig |
| [`scafera/integration`](integration.html) | External APIs — Gateway pattern, `HttpClient` |
| [`scafera/log`](log.html) | Logging — PSR-3 JSON Lines, CLI filters |

### Tooling

| Package | Purpose |
|---------|---------|
| [`scafera/scaffold`](scaffold.html) | Composer plugin — scaffolds framework-owned files |

### Project templates

| Package | Purpose |
|---------|---------|
| `scafera/layered-web` | `composer create-project` starter for layered web apps |

---

## For AI agents

If you are an AI agent working in a Scafera project:

- Start with **[conventions](conventions.html)** — the hard rules you must not break
- The project's `AGENTS.md` (or `CLAUDE.md`) contains project-specific excerpts of these rules
- A single-file machine-readable reference is available at [`llms-full.txt`](llms-full.txt)
- Before declaring work done, run `vendor/bin/scafera validate` — violations are build failures, not suggestions

---

## Source

- GitHub org: [github.com/scafera](https://github.com/scafera)
- Each package is its own repository under the `scafera/` organization
