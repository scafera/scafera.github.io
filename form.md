---
layout: page
title: Forms
permalink: /form/
---

# scafera/form

Form handling and validation. DTO-based forms with attribute validation, CSRF protection, typed value handling.

Internally adopts `symfony/form`, `symfony/validator`, and `symfony/security-csrf`. Userland never imports Symfony Form or Validator types — boundary enforcement blocks it at compile time.

---

## Install

```bash
composer require scafera/form
```

---

## Core ideas

- **DTOs + attributes, not Symfony FormType** — plain PHP classes with `Rule\*` attributes for the common case
- **CSRF scoped per DTO class** — a token from one form cannot submit another
- **File uploads handled separately** — use `scafera/file` alongside `FormHandler`
- **`src/Form/` is a controlled zone** — Symfony Form types allowed there only, same pattern as `src/Repository/` for Doctrine
- **Type transformation on old values** — `old()` returns typed values (int, enum, null), not raw strings

---

## Input DTO

```php
use Scafera\Form\Rule;

final class CreatePostInput
{
    #[Rule\NotBlank]
    #[Rule\MaxLength(200)]
    public string $title = '';

    #[Rule\MaxLength(5000)]
    public ?string $body = null;

    #[Rule\Positive]
    public int $priority = 1;
}
```

Supported property types: `string`, `int`, `float`, `bool`, `DateTimeImmutable`, nullable variants, `BackedEnum`.

---

## Available rules

| Rule | Parameters |
|------|-----------|
| `#[NotBlank]` | — |
| `#[MaxLength(int)]` | max characters |
| `#[MinLength(int)]` | min characters |
| `#[Min(int\|float)]` | minimum value |
| `#[Max(int\|float)]` | maximum value |
| `#[Positive]` | must be > 0 |
| `#[OneOf(array)]` | allowed values |
| `#[Email]` | email format |
| `#[FileExtension(array)]` | allowed file extensions |

---

## Standalone validation (API path)

```php
use Scafera\Form\Validator;

$input = new CreatePostInput();
$input->title = '';

$result = $validator->validate($input);

$result->hasErrors();      // true
$result->firstError();     // ValidationError{field: 'title', message: '...'}
$result->toArray();        // ['title' => ['This value should not be blank.']]
```

---

## Form handling (web path)

```php
use Scafera\Form\FormHandler;
use Scafera\Kernel\Contract\ViewInterface;
use Scafera\Kernel\Http\{Request, Response, RedirectResponse, Route};

#[Route('/posts/new', methods: ['GET', 'POST'])]
final class CreatePost
{
    public function __construct(
        private readonly FormHandler $form,
        private readonly ViewInterface $view,
    ) {}

    public function __invoke(Request $request): Response
    {
        $form = $this->form->handle($request, CreatePostInput::class);

        if ($form->isValid()) {
            $input = $form->getData();           // typed CreatePostInput
            // persist...
            return new RedirectResponse('/posts');
        }

        return new Response($this->view->render('posts/new.html.twig', [
            'form' => $form,
        ]));
    }
}
```

---

## Template

Plain HTML. No form themes, no `form_widget()`:

{% raw %}
```twig
<form method="POST">
    <input type="hidden" name="_csrf" value="{{ form.csrfToken() }}">

    <label>Title</label>
    <input name="title" value="{{ form.old('title') }}">
    {% if form.error('title') %}
        <span class="error">{{ form.error('title') }}</span>
    {% endif %}

    <button type="submit">Create</button>
</form>
```
{% endraw %}

---

## `Form` API

| Method | Returns |
|--------|---------|
| `isSubmitted()` | `bool` |
| `isValid()` | `bool` (true only when submitted + valid) |
| `getData()` | typed input DTO |
| `errors()` | `array<string, list<string>>` |
| `error(string $field)` | `?string` (first error for field) |
| `old(string $field)` | `mixed` (submitted value, type-transformed) |
| `csrfToken()` | `string` |

---

## Controlled zone: `src/Form/`

For forms that exceed DTO capabilities (nested forms, dynamic fields, entity-backed choices), place Symfony `AbstractType` classes in `src/Form/`:

```
src/
  Form/            Symfony Form imports allowed here
    OrderForm.php
```

**Rules for `src/Form/`:**

- May reference `App\Entity` for type hints and structure — must NOT perform persistence or lifecycle operations
- Must NOT import from `App\Service`, `App\Repository`, `App\Controller`, `App\Command`, or `App\Integration`
- Only controllers may import from `App\Form` — services must NOT depend on forms
- You wire Form classes into your controller via Symfony's `FormFactory` directly

---

## Boundary enforcement

| Blocked | Use instead |
|---------|-------------|
| `Symfony\Component\Form\*` | `Scafera\Form\FormHandler` |
| `Symfony\Component\Validator\*` | `Scafera\Form\Validator` + `Rule\*` |

Allowed inside `src/Form/` only. Enforced via compiler pass (build time) and validator (`scafera validate`). Detects `use`, `new`, and `extends` patterns.
