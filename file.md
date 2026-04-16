---
layout: page
title: Files
permalink: /file/
---

# scafera/file

File handling for Scafera. Upload, validate, store, and serve files — all behind Scafera-owned types.

Internally adopts `symfony/http-foundation` Upload, `symfony/mime`, `symfony/filesystem`. Userland never imports Symfony file types — boundary enforcement blocks it at compile time.

---

## Install

```bash
composer require scafera/file
```

---

## Core ideas

- **MIME detection is server-side** — uses magic bytes from `symfony/mime`, not the client-supplied `Content-Type` header. Prevents spoofed file types.
- **Path traversal is structurally prevented** — directory segments sanitized (stripped of `..` and `.`) before any filesystem operation. No file is ever written before validation.
- **Collision handling** — if a file with the same name exists, a unique random suffix is appended. No silent overwrite.
- **Storage path is architecture-owned** — defined by `getStorageDir()` on the architecture package. For `scafera/layered`: `var/uploads/`.
- **Form and file are separate capabilities** — file uploads are NOT handled by `scafera/form`. Controllers compose both packages explicitly.

---

## Upload and validate

```php
use Scafera\File\UploadExtractor;
use Scafera\File\UploadValidator;
use Scafera\File\UploadConstraint;

$file = $uploads->get($request, 'avatar');

if ($file !== null) {
    $result = $validator->validate($file, new UploadConstraint(
        allowedExtensions: ['jpg', 'png'],
        allowedMimeTypes: ['image/jpeg', 'image/png'],
        maxSizeBytes: 2_097_152,          // 2 MB
    ));

    if (!$result->isValid()) {
        // $result->error() — human-readable message
    }
}
```

---

## Store

```php
use Scafera\File\FileStorage;

$path = $storage->store($file, 'avatars');               // 'avatars/photo.jpg'
$path = $storage->store($file, 'avatars', 'me.jpg');     // 'avatars/me.jpg'

$storage->exists($path);       // true
$storage->delete($path);       // true
```

The storage directory is defined by the architecture package. For `scafera/layered`: `var/uploads/`. Filenames are sanitized (directory components stripped). If a file with the same name exists, a unique suffix is appended automatically.

---

## Serve

```php
use Scafera\File\FileResponse;

return FileResponse::download($path, 'report.pdf');
return FileResponse::inline($path);
```

`FileResponse` does not implement `ResponseInterface`. A dedicated listener converts it to a binary response at priority 10 (before the kernel's response listener).

---

## File uploads in forms

File uploads are **not** handled by `scafera/form`. Use both packages together in your controller:

```php
$form = $this->formHandler->handle($request, ProfileInput::class);
$avatar = $this->uploads->get($request, 'avatar');
```

Two explicit calls. Form handles POST data. File handles uploads. Each validates independently.

---

## Boundary enforcement

| Blocked | Use instead |
|---------|-------------|
| `Symfony\Component\HttpFoundation\File\UploadedFile` | `Scafera\File\UploadedFile` |
| `Symfony\Component\HttpFoundation\File\File` | `Scafera\File\FileStorage` |
| `Symfony\Component\Filesystem\*` | `Scafera\File\FileStorage` |

Enforced via compiler pass (build time) and validator (`scafera validate`).
