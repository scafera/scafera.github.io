---
layout: page
title: Auth
permalink: /auth/
---

# scafera/auth

Authentication and access control. Session management, guards, login/logout, and password hashing — all behind Scafera-owned types.

Internally adopts `symfony/http-foundation` Session and `symfony/password-hasher`. Userland never imports Symfony types — boundary enforcement blocks it at compile time.

---

## Install

```bash
composer require scafera/auth
```

---

## What it provides

- `Session` — session state with flash messages
- `CookieJar` — secure cookie handling (auto-applied via response listener)
- `Authenticator` — login, logout, user resolution
- `Password` — hash, verify, `needsRehash`
- `GuardInterface` + `#[Protect]` — route protection
- `SessionGuard` and `RoleGuard` — built-in guards
- `User` + `UserProvider` — user identity contracts

---

## Design decisions

- **User existence is the source of truth** — `isAuthenticated()` verifies the user still exists in the provider, not just that a session key is present. One cached DB lookup per request.
- **Session fixation prevention** — session ID regenerated on both login and logout.
- **Explicit guard execution** — guards declared via `#[Protect]` on controllers, not via implicit firewall rules. Options passed directly to `check()` — no magic attributes.

---

## Session

```php
use Scafera\Auth\Session;

$session->set('key', 'value');
$session->get('key');                    // 'value'
$session->has('key');                    // true
$session->remove('key');
$session->flash('notice', 'Saved!');
$session->getFlash('notice');            // ['Saved!']
```

Safe in CLI context — returns defaults when no request exists.

---

## Login / logout

```php
use Scafera\Auth\Authenticator;
use Scafera\Auth\Password;
use Scafera\Auth\UserProvider;

$user = $userProvider->findByIdentifier($email);

if ($user && $password->verify($user->getPassword(), $plainPassword)) {
    $authenticator->login($user);        // regenerates session ID
}

$authenticator->isAuthenticated();       // verifies user exists in provider
$authenticator->getUser();               // User or null

$authenticator->logout();                // removes session key, regenerates session ID

if ($password->needsRehash($user->getPassword())) {
    // update stored hash
}
```

---

## User contracts

Implement these in your application:

```php
use Scafera\Auth\User;
use Scafera\Auth\UserProvider;

final class AppUser implements User
{
    public function getIdentifier(): string { /* ... */ }
    public function getRoles(): array { /* ... */ }
    public function getPassword(): string { /* ... */ }
}

final class AppUserProvider implements UserProvider
{
    public function findByIdentifier(string $identifier): ?User { /* ... */ }
}
```

When exactly one `UserProvider` implementation exists, it is auto-aliased for injection.

---

## Route protection

```php
use Scafera\Auth\Protect;
use Scafera\Auth\SessionGuard;
use Scafera\Auth\RoleGuard;
use Scafera\Kernel\Http\Route;

#[Route('/profile', methods: ['GET', 'POST'])]
#[Protect(guard: SessionGuard::class)]
final class EditProfile
{
    // Only authenticated users reach here — others redirect to /login
}

#[Route('/admin', methods: 'GET')]
#[Protect(guard: RoleGuard::class, options: ['role' => 'ADMIN'])]
final class AdminDashboard
{
    // Only users with ADMIN role
}
```

Guards run in declaration order. Return `null` to allow, or a `ResponseInterface` to deny. Options from `#[Protect]` pass directly to `check()` — no magic attributes.

---

## Global guards

```yaml
# config/config.yaml
scafera_auth:
    global:
        - App\Guard\MaintenanceGuard
    exclude:
        - /health
        - /login
```

Global guards run before route-specific guards. Excluded paths match exactly or as prefixes with `/`.

---

## Custom guard

```php
use Scafera\Auth\GuardInterface;
use Scafera\Kernel\Http\Request;
use Psr\Http\Message\ResponseInterface;

final class MaintenanceGuard implements GuardInterface
{
    public function check(Request $request, array $options = []): ?ResponseInterface
    {
        if ($this->inMaintenance()) {
            return new RedirectResponse('/maintenance');
        }

        return null;
    }
}
```

---

## Boundary enforcement

| Blocked | Use instead |
|---------|-------------|
| `Symfony\Component\HttpFoundation\Session\*` | `Scafera\Auth\Session` |
| `Symfony\Component\HttpFoundation\Cookie` | `Scafera\Auth\CookieJar` |
| `Symfony\Component\Security\*` | `Scafera\Auth\Authenticator`, `GuardInterface` |
| `Symfony\Component\PasswordHasher\*` | `Scafera\Auth\Password` |

Enforced via compiler pass (build time) and validator (`scafera validate`). Detects `use`, `new`, and `extends` patterns.
