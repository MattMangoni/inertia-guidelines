# Inertia (and DTOs) Guidelines

## Context (Project-Specific)

- Framework: Laravel 13 + Inertia v2
- PHP: 8.5
- DTO package: `spatie/laravel-data`
- Goal: enforce end-to-end typing from backend payloads to frontend TypeScript.

## Non-Negotiable Rules

- Every prop passed to Inertia must be typed through DTOs.
- Never pass raw Eloquent models or raw Eloquent collections directly to Inertia props.
- All DTO creation must use `::from(...)`.
- Never instantiate DTOs with `new` in application code.
- For collections of DTOs, use `Spatie\LaravelData\DataCollection`.
- Keep backend DTO shape and frontend TypeScript shape aligned via the TypeScript transformer output.

## Helper-First Laravel Style

- Prefer Laravel helpers over static facades whenever a helper exists and keeps intent clear.
- For Inertia responses, prefer `inertia(...)` helper over `Inertia::render(...)`.
- Use facade/static calls only when no helper exists or when explicitly required.

## Model Key Access

- Always use `->getKey()` when reading a model primary key.
- Never use `->id` directly for primary key access.
- This applies in DTO mapping, controllers, actions, policies, and business logic.

## Backend Inertia Contract Rules

- In every Inertia response, ensure props are DTOs (or primitives), not models.
- When a page DTO represents the full Inertia props payload, pass the DTO directly as the second `inertia(...)` argument. Do not wrap it in an array with `...Dto::from([...])->toArray()`.
- Shared props in `app/Http/Middleware/HandleInertiaRequests.php` must be DTO-backed where structured data is returned.
- Nested payloads should use nested DTOs rather than untyped associative arrays when structure is stable.

## DTO Authoring Rules

- Place DTOs in `app/Data`.
- DTOs should extend `Spatie\LaravelData\Data`.
- Use explicit typed properties for all fields.
- Use `::from(...)` everywhere DTOs are created.
- When mapping from Eloquent is non-trivial, add a typed `fromModel(...)` method.
- Calling `new self(...)` is allowed only inside the DTO class (for example in `fromModel(...)`).
- Never call `new DtoClass(...)` directly outside the DTO class.

## DTO Collection Rules

- Collections of DTOs must be `DataCollection`.
- Prefer:
    - `UserData::collect($users, \Spatie\LaravelData\DataCollection::class)`
- Do not pass `Collection<Model>` directly to Inertia.

## TypeScript Transformer Alignment

- DTOs used in frontend payloads must be included in the TypeScript transformer flow.
- Frontend types should reference generated DTO-based types (or types derived from generated output), not duplicated hand-written types where avoidable.
- When DTO shape changes, regenerate/update TS types before merging.

## Frontend Typing Expectations

- Page props should be fully typed and mirror DTO contracts exactly.
- Avoid `any`.
- Avoid broad `[key: string]: unknown` unless the data is truly dynamic.
- Keep `resources/js/types` aligned with backend DTO contracts until generated types are consumed directly everywhere.

## Example DTO Pattern

```php
<?php

declare(strict_types=1);

namespace App\Data;

use App\Models\User;
use Spatie\LaravelData\Data;

final class UserData extends Data
{
    public function __construct(
        public int $id,
        public string $name,
        public string $email,
    ) {
    }

    public static function fromModel(User $user): self
    {
        return new self(
            id: $user->getKey(),
            name: $user->name,
            email: $user->email,
        );
    }
}
```

Usage in controller / middleware:

```php
return inertia('Dashboard', DashboardPageData::from([
    'user' => UserData::from($request->user()),
]));
```

Collection usage:

```php
return inertia('Users/Index', [
    'users' => UserData::collect($users, \Spatie\LaravelData\DataCollection::class),
]);
```

## Progressive Migration Rule

- When touching existing code, normalize it to these standards:
    - Replace `->id` with `->getKey()`.
    - Replace `Inertia::render(...)` with `inertia(...)`.
    - Replace raw Inertia payload arrays/models with DTOs.
    - Replace manual DTO instantiation with `::from(...)`.
