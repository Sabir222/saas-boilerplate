# Goals

## Goal 1 - Replace Clerk with Better Auth

### Why this is the first goal

The OSS starter still ships with Clerk baked into routing, pages, env validation, and dashboard UX. To move the project closer to Max, auth needs to become app-owned and self-hostable.

### Current project analysis

#### Database

- The app uses PostgreSQL with Drizzle ORM through `pg` and `pglite-server`.
- The shared DB connection already exists in `src/utils/DBConnection.ts` and `src/libs/DB.ts`.
- The schema is still minimal: `src/models/Schema.ts` only defines the `counter` table.
- There are no app-owned auth tables yet for users, sessions, accounts, or verification tokens.

#### Clerk setup today

- `@clerk/nextjs` and `@clerk/localizations` are installed in `package.json`.
- `src/libs/Env.ts` requires `CLERK_SECRET_KEY` and `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`.
- `src/proxy.ts` uses Clerk middleware plus `auth.protect()` to guard `/dashboard` and auth routes.
- `src/app/[locale]/(auth)/layout.tsx` wraps auth pages in `ClerkProvider` and handles localized Clerk URLs.
- `src/app/[locale]/(auth)/(center)/sign-in/[[...sign-in]]/page.tsx` renders Clerk `SignIn`.
- `src/app/[locale]/(auth)/(center)/sign-up/[[...sign-up]]/page.tsx` renders Clerk `SignUp`.
- `src/app/[locale]/(auth)/dashboard/layout.tsx` uses Clerk `SignOutButton`.
- `src/app/[locale]/(auth)/dashboard/user-profile/[[...user-profile]]/page.tsx` uses Clerk `UserProfile`.
- `src/components/Hello.tsx` reads the signed-in user through Clerk `currentUser()`.
- `src/utils/AppConfig.ts` and `src/styles/global.css` include Clerk-specific localization and styling glue.
- README and marketing content still describe Clerk as the auth solution.

### Better Auth docs notes that affect this migration

- Better Auth recommends a shared server auth instance plus a Next.js route handler mounted at `/api/auth/[...all]` via `toNextJsHandler(auth)`.
- The client side should use a dedicated auth client created with `createAuthClient()`.
- Server-side session reads should use `auth.api.getSession({ headers: await headers() })`.
- Better Auth and Next.js docs both recommend treating `proxy.ts` as an optimistic redirect layer, not the primary authorization boundary.
- The Clerk migration guide says active Clerk sessions will be invalidated.
- Clerk password hashes are bcrypt-based, so Better Auth must be configured to verify bcrypt hashes during migration.
- The Clerk migration guide covers users, accounts, passwords, email verification, phone numbers, and 2FA data.
- The Clerk migration guide does not cover organizations out of the box; if teams/orgs are added later, we should plan for the Better Auth organization plugin instead of doing a second auth redesign.

### Best migration plan

#### Phase 1 - Add Better Auth foundation

- Add Better Auth packages and create a shared auth module, ideally `src/libs/Auth.ts` plus `src/libs/AuthClient.ts`.
- Reuse the existing Drizzle/PostgreSQL connection instead of introducing a second DB layer.
- Add the Better Auth route handler at `src/app/api/auth/[...all]/route.ts`.
- Extend `src/libs/Env.ts` with the Better Auth env values needed for secret, base URL, and any enabled providers.

#### Phase 2 - Add app-owned auth schema

- Add Better Auth tables to Drizzle for at least `user`, `session`, `account`, and `verification`.
- Keep the existing `counter` table untouched.
- Generate a migration so local PGlite and production PostgreSQL stay aligned.
- If we know we will need username, phone, admin, 2FA, or organization support soon, decide those plugins before generating the final auth schema.

#### Phase 3 - Replace Clerk UI with app-owned pages

- Replace Clerk `SignIn` and `SignUp` pages with local forms wired to Better Auth.
- Replace Clerk `SignOutButton` with a local sign-out action.
- Replace Clerk `UserProfile` with an app-owned account/settings page.
- Keep the current locale-aware route structure and `next-intl` metadata behavior.
- Add any missing translations instead of hard-coding new auth copy.

#### Phase 4 - Move auth checks to the app layer

- Replace `currentUser()` usage with Better Auth session reads in server components.
- Add a small server-side auth helper or DAL so pages and route handlers can consistently fetch and verify the session.
- Keep `src/proxy.ts` only for fast redirects between public auth pages and `/dashboard`.
- Do secure authorization checks in server pages, server actions, and route handlers rather than relying on proxy alone.

#### Phase 5 - Migrate Clerk data

- Stand up Better Auth first and verify new sign-up/sign-in works on a fresh database.
- Export Clerk users and adapt the Better Auth Clerk migration script for this repo.
- Preserve Clerk user IDs where possible so downstream data can keep stable references.
- Migrate bcrypt password hashes, social accounts, email verification state, and any 2FA data we decide to support.
- Accept that users will need to sign in again after cutover because Clerk sessions cannot be preserved.

#### Phase 6 - Remove Clerk completely

- Delete Clerk dependencies, env vars, provider wrappers, localization helpers, CSS layer naming, and profile widgets.
- Remove Clerk references from README, marketing copy, sponsor assets, and setup docs.
- Re-run lint, types, tests, and build-local after the cutover.

### Recommended implementation order inside this repo

1. Land Better Auth route handler, auth instance, client instance, env updates, and DB schema.
2. Ship local sign-in, sign-up, sign-out, and dashboard session checks.
3. Replace the Clerk user profile page with a basic account page.
4. Only then perform Clerk data migration and final dependency cleanup.

### Expected challenges

- Clerk currently provides hosted auth UI, while Better Auth shifts more UI ownership into the app.
- The current database has no auth tables yet, so auth is both a product migration and a schema expansion.
- `proxy.ts` currently mixes Arcjet, i18n, and Clerk protection, so the replacement needs to preserve those concerns without reintroducing heavy auth logic there.
- Social login, password reset, email verification, and account management all become explicit app responsibilities.

### Definition of done

- Clerk is fully removed from runtime code, dependencies, env validation, and docs.
- Better Auth owns sign-in, sign-up, sign-out, session reads, and dashboard protection.
- The database contains the required Better Auth tables and migrations.
- Existing locale-aware auth routes still work.
- The repo passes `bun run lint`, `bun run check:types`, `bun run test`, and `bun run build-local`.

## Goal 2 - Add multi-tenancy and teams

Details to be defined when work starts.

## Goal 3 - Add RBAC

Details to be defined when work starts.

## Goal 4 - Add Stripe billing

Details to be defined when work starts.

## Goal 5 - Add oRPC

Details to be defined when work starts.

## Goal 6 - Add Shadcn UI foundation

Details to be defined when work starts.
