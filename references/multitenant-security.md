# Multi-tenant security checklist (Next.js + Supabase)

Read this before shipping any new table, RPC, Server Action, Route Handler, storage bucket or permission change, and when asked for a security review. In a multi-tenant SaaS the worst bug isn't a crash — it's tenant A seeing, or changing, tenant B's data. Every item here is a way that happens in this stack. The built-in `/security-review` command is a good second pass; this checklist is the stack-specific first one.

The principle behind all of it (gotchas #3): **the database is the tenant boundary.** Application code, hidden buttons and middleware are convenience; RLS and checks inside the database or the server are the actual wall.

## 1. Never trust the tenant, user or role from the client

- **`empresa_id` comes from the session, never from input.** A Server Action or RPC that accepts `empresa_id` as a parameter and uses it to scope a query is a cross-tenant hole: change the value in the request, read another tenant. Derive it with `current_empresa_id()` in SQL, or from the authenticated user on the server.
- **Ids in URLs and forms are just suggestions.** `/ventas/123` or `anularVenta(id)` must still be checked against the caller's tenant (RLS does this if the query runs as the user; see section 3 for when it doesn't).
- **Role/level comes from your tables or `app_metadata`, never `user_metadata`.** Supabase lets the user edit their own `user_metadata` — authorizing on it lets anyone promote themselves.
- **On the server, authenticate with `supabase.auth.getUser()`** (or verified claims), which validates the token with Supabase Auth. Don't authorize on the result of `getSession()` read from cookies on the server — that data isn't revalidated.

## 2. Every Server Action and Route Handler is a public endpoint

A `"use server"` function can be invoked by anyone with a POST request and arbitrary arguments — hiding the button doesn't remove the endpoint. Each one, at the top:

1. Authenticates the caller (no user → return `{ ok: false, error }`, per gotchas #1).
2. Checks the module and, for high-stakes actions, the level (voiding a sale, changing prices or commissions, managing users → owner-level check, per `multitenant-architecture.md`).
3. Validates the input shape (zod or equivalent) — types from TypeScript don't exist at runtime.
4. Runs the query **as the user** (the server client carrying the user's session), so RLS applies.

Middleware is a routing convenience, not an authorization layer — it can be misconfigured, skipped by a matcher, or bypassed (Next.js shipped a middleware-bypass vulnerability in 2025). Check authorization where the data is accessed, too.

**Audit it the way `SKILL.md` says:** grep every `"use server"` file and every `route.ts` and inventory which ones skip auth, skip the role check, or accept `empresa_id` from input. One missing check is usually a template that got copied.

## 3. Things that bypass RLS — know each one

| Bypass | Risk | Rule |
|---|---|---|
| **`service_role` key** | Full access to every tenant | Server-only, never in a `NEXT_PUBLIC_*` variable or a client bundle. Use it only for genuinely cross-tenant jobs (cron, webhooks), and scope every query by `empresa_id` explicitly inside them. Grep the built client bundle for the key if in doubt. |
| **`security definer` functions** | Run as the function owner, ignoring the caller's RLS | Check tenant and permission *inside* the function (`where empresa_id = current_empresa_id()`, `current_tiene_modulo(...)`, `current_nivel()`). Set `set search_path = ''` and schema-qualify names. Prefer `security invoker` unless you need definer. |
| **Views** | By default run with the view owner's rights, not the caller's | Create them with `with (security_invoker = true)` so the underlying tables' RLS applies. A plain view over a tenant table can expose every tenant. |
| **Function execute grants** | New functions in `public` are executable by `anon` and `authenticated` by default | For internal helpers and admin-only RPCs: `revoke execute on function … from public, anon;` then grant only what's needed. |
| **Table owner / superuser** | Bypass RLS unless forced | Relevant for local testing: tests run as `postgres` prove nothing about RLS. Test as real `authenticated` users (section 6). |

## 4. RLS policies: complete, consistent, tenant-first

For every table holding tenant data:

- [ ] `enable row level security` is on (Supabase's security advisor flags tables without it — keep it at zero findings).
- [ ] Policies cover each command the app uses: `select`, `insert`, `update`, `delete`. A missing `delete` policy means nobody can delete; an overly broad one means anyone in the tenant can.
- [ ] `insert` and `update` have `with check (empresa_id = current_empresa_id() …)`, not only `using`. Without `with check`, a user can write a row into another tenant, or move a row out of theirs by updating `empresa_id`.
- [ ] Every policy starts from `empresa_id = current_empresa_id()`; module/level conditions are added on top, never instead.
- [ ] The gating pattern matches sibling tables and the migration that set it — don't "restore" a role check someone removed on purpose (`multitenant-architecture.md`).
- [ ] Child tables that inherit their tenant through a parent (e.g. `detalle_pedido` via `pedidos`) either carry their own `empresa_id` or check it via the parent in the policy — never "the parent is protected, so the child must be."

## 5. Storage, realtime and other side doors

- **Storage buckets:** private by default. A public bucket's files are readable by anyone with the URL — fine for a product photo on a public menu, never for contracts, IDs or receipts. Store objects under an `empresa_id/…` path prefix and write `storage.objects` policies that check it (`(storage.foldername(name))[1] = current_empresa_id()::text`). Serve private files with short-lived signed URLs.
- **Realtime:** database-change subscriptions respect RLS; broadcast/presence channels need their own authorization. Never name a channel with a guessable tenant id and assume it's private.
- **Webhooks** (payments, integrations): verify the signature before trusting the payload; they run with the service role, so resolve the tenant from the verified payload, not from a query parameter.
- **Emails, exports, PDFs, reports:** generated server-side with elevated rights more often than screens are — check they filter by the requesting tenant.
- **Error messages:** returning `error.message` (gotchas #1) is right for business-rule errors; don't forward raw SQL errors that leak table names or other tenants' values — map them to a user-facing message.

## 6. Test the boundary, don't assume it

Integration tests that run as real `authenticated` users, with **at least two tenants** seeded:

- A user of tenant A cannot `select`, `update` or `delete` a row of tenant B — expect zero rows or an error, not "the UI didn't show it."
- A user cannot `insert` a row with another tenant's `empresa_id`, or `update` a row's `empresa_id` to another tenant.
- For each gated module: a user without the module is denied; a user granted it by per-user override (not by role) is allowed.
- High-stakes actions reject non-owner levels even when the module is granted.
- Every RPC that's `security definer` gets the same cross-tenant test as a table.

These are the tests that caught the reintroduced-role-check regression in `multitenant-architecture.md` — review missed it. When you add a table or RPC, add its cross-tenant test in the same change.

## 7. Secrets and configuration

- Only values that are genuinely public get the `NEXT_PUBLIC_` prefix (the Supabase URL and anon key are designed to be public; RLS is what protects the data).
- Secrets live in the hosting provider's environment settings, never in the repo. If one was ever committed, rotate it — deleting the commit doesn't un-leak it.
- Separate keys per environment; a preview deployment must not hold production's service role key unless it truly needs it.
- Auth settings: email confirmation on, sensible password rules, rate limits on sign-in and password reset, redirect URLs restricted to your own domains.

## Quick review before merging

- [ ] New tables: RLS on, all needed policies with `with check`, `empresa_id not null` + index.
- [ ] New RPCs: tenant/permission checks inside if `security definer`; execute grants reviewed; old signatures dropped (gotchas #2).
- [ ] New Server Actions / routes: auth, permission, input validation at the top; no `empresa_id` from input; `{ok, error}` returns.
- [ ] No `service_role` usage outside server-only, cross-tenant code.
- [ ] New views use `security_invoker`.
- [ ] Storage paths prefixed by tenant, bucket private unless it's genuinely public content.
- [ ] Cross-tenant tests added for everything new.
- [ ] Supabase security advisor: no new findings.
