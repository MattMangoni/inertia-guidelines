# React + Inertia Frontend Guidelines

## Context (Project-Specific)

- Stack: Inertia v3 + React 19 + TypeScript strict + Tailwind v4
- Routes/actions typed via Laravel Wayfinder (`@/routes`, `@/actions`)
- Backend DTOs export to `App.Data.*` namespace via `spatie/typescript-transformer`
- See also: `inertia-dto-guidelines.md` for backend DTO/payload rules

## Non-Negotiable Rules

- All backend-shape TS types come from generated `App.Data.*`. Never hand-write a TS type for anything a controller can return.
- All navigation uses Wayfinder typed functions. Never hard-code URLs in `<Link>`/`router.visit`.
- Pages live in `resources/js/pages/<kebab-case>.tsx` and use a single default export named in PascalCase matching the file.
- `resources/js/types/global.d.ts` must keep its `export {};` and the `[key: string]: unknown` inside `sharedPageProps`. Both are load-bearing — see comments in the file.
- Never edit `resources/js/types/generated.d.ts` by hand. It's regenerated and gitignored.
- Don't add `useMemo` / `useCallback` / `React.memo` defensively — React Compiler handles memoization. Add only after profiling proves a need.

## File Layout

- Pages: `resources/js/pages/<kebab-case>.tsx` ← matches `inertia('welcome')` → `welcome.tsx`
- Reusable components: `resources/js/components/`
- Page-local components: co-locate under `resources/js/pages/<page>/_components/`
- Hooks: `resources/js/hooks/`
- Helpers: `resources/js/lib/`
- Path alias: `@/*` → `./resources/js/*`. No `../../` traversal.

## Typing Shared & Page Props

- Add new shared props in **two** places: `HandleInertiaRequests::share()` (backend) *and* the `InertiaConfig.sharedPageProps` augmentation in `global.d.ts` (frontend). Use `App.Data.*` types in the augmentation — never re-declare those shapes.
- Per-page props: declare a local `type Props = {…}` in the page file composed from `App.Data.*` types, then `usePage<Props>().props`.

```tsx
// resources/js/pages/users/show.tsx
import { usePage } from '@inertiajs/react';

type Props = {
    user: App.Data.UserData;
    posts: App.Data.PostData[];
};

export default function ShowUser() {
    const { user, posts, auth, name } = usePage<Props>().props;
    // user, posts: page-specific (from controller)
    // auth: App.Data.AuthData, name: string ← from shared props
}
```

## Generated Types Workflow

- After adding/changing a `#[TypeScript]`-marked DTO: `composer types:generate` (or `npm run types:generate`).
- `composer dev` runs the transform on startup, so cold-start gets fresh types.
- Hand-written types in `resources/js/types/` are reserved for **purely-frontend** concepts (UI state, 3rd-party config). Anything from the backend lives as a DTO.

## Routes & Actions (Wayfinder)

- Import typed functions, never strings:
  ```tsx
  import { home } from '@/routes';
  import UserController from '@/actions/App/Http/Controllers/UserController';

  <Link href={home()}>Home</Link>
  <Form action={UserController.store.form()}>…</Form>
  ```
- `route('home')` (Laravel helper) is fine **only** in Pest tests.
- Regen via `php artisan wayfinder:generate` (also runs as a Vite plugin).

## Navigation

- `<Link>` from `@inertiajs/react` for in-app navigation
- `router.visit(...)` for programmatic navigation
- `<a>` only for external links / downloads
- For data mutations: prefer `<Form>` + `useForm` over manual `fetch`

## Forms

- `useForm` owns state, errors, processing, dirty tracking. Don't re-implement with `useState`.
- Validation errors from backend `FormRequest`s arrive automatically via Inertia — don't poll or re-fetch.

## Standalone HTTP (v3)

- For non-Inertia requests, use the `useHttp` hook or the built-in XHR client.
- **Do not install or import `axios`** — Inertia v3 removed it.
- Renamed events: `httpException` (was `invalid`), `networkError` (was `exception`). `router.cancelAll()` replaces `router.cancel()`.

## v3 Prop Strategies

- Deferred props: **always pair with a pulsing/animated skeleton** placeholder. Never leave UI empty.
- Optional: `Inertia::optional()` (v3). `Inertia::lazy()` is removed.
- Merging: `Inertia::merge()` for incremental loads.
- Hot paths: `<Link prefetch>` for instant visits.

## React 19 + Compiler

- React Compiler is on. Idiomatic top-down props are fine — re-render cost is handled.
- Avoid `useEffect` for derived state. Compute during render.
- Skip the `useMemo`/`useCallback` reflex. Reach for them only after measuring.

## Styling

- Tailwind v4 utility classes. No one-off CSS files — extract a component instead.
- Conditional/merged classes via `cn(...)` from `@/lib/utils` (`tailwind-merge` + `clsx`).
- Variant-driven components: `class-variance-authority` (already in deps).
- Dark mode: `dark:` variants. No theme provider.

## Imports & Style

- Prettier owns formatting; don't fight it.
- ESLint owns import order and React rules. Don't `eslint-disable` without a one-line justification.
- React event types: `ChangeEvent<HTMLInputElement>`, `MouseEvent<HTMLButtonElement>`, etc. No `any`.

## Anti-Patterns

- Hand-writing TS types for backend shapes (use DTOs)
- Forgetting `composer types:generate` after a DTO change
- Passing Eloquent models / raw arrays to `inertia(...)` (see `inertia-dto-typing.md`)
- Inline route closures for anything beyond trivial single-line redirects (use invokable controllers)
- Hard-coded URLs in `<Link>` / `router.visit` (use Wayfinder)
- Removing `export {};` or the `[key: string]: unknown` from `global.d.ts`
- Manual memoization layers added before profiling
- Re-installing `axios` (use `useHttp` / built-in XHR)
- Editing `resources/js/types/generated.d.ts` by hand
- Mixing v2 Inertia API (`Inertia::lazy`, `router.cancel`, axios) with v3

## Skills to Activate

- `inertia-react-development` — for any Inertia React patterns
- `tailwindcss-development` — for utility-class work
- `wayfinder-development` — for route/action wiring
- `pest-testing` — for tests touching the React tree
