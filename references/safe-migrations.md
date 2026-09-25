# Safe schema migrations (Postgres / Supabase)

Read this before writing or applying any migration against a database that real tenants are using: a new table, a new or changed column, a changed RPC signature, a new RLS policy, an index, or a data backfill. A bad deploy of app code can be rolled back in seconds; a bad migration can lock a table mid-shift, break every open browser tab, or destroy data you can't get back. Treat them as different weight classes.

## The core constraint: two app versions share one schema

During and after a deploy, the schema must work with **both** the old and the new app code at the same time:

- Vercel serves the new build immediately, but browsers with the old bundle stay open — a cashier's POS tab can run yesterday's JavaScript for hours, calling yesterday's Server Actions and RPC signatures.
- The migration and the deploy never land at exactly the same instant. Whichever goes first, the other version sees a schema it wasn't written for.

So any change that removes or renames something the old code uses is a breaking change, even if the new code is perfect. The fix is always the same shape: **expand, migrate, contract** — never in one step.

## Expand → migrate → contract

| Step | What it does | Safe because |
|---|---|---|
| **1. Expand** (migration) | Add the new thing alongside the old: new column (nullable or with a constant default), new table, new function signature | Old code ignores it |
| **2. Deploy app** | New code writes to both old and new (or reads new with a fallback to old) | Both versions keep working |
| **3. Backfill** (migration or script) | Copy/derive existing data into the new shape, in batches | Nobody depends on it being complete yet |
| **4. Switch** (deploy) | New code reads and writes only the new thing | Old column still exists for stragglers |
| **5. Contract** (migration, days later) | Drop the old column/function/policy | No running code references it anymore — verify with a grep, not memory |

Common cases:

- **Rename a column** → never `ALTER TABLE … RENAME COLUMN` on a live table. Add the new column, dual-write, backfill, switch reads, drop the old one later.
- **Add a parameter to an RPC** → the old signature must keep working for open tabs. Either add the parameter *with a default* and drop the old signature in the same migration (see gotchas #2 — `create or replace` with a different arity makes an overload, not a replacement), or keep both signatures alive until the contract step.
- **Change a column's type** → add a new column of the new type; a type change rewrites the table under an exclusive lock.
- **Make a column required** → backfill first, then add the constraint (see "Locks" below). Adding `NOT NULL` to a column that still has nulls fails; adding it to a large table blocks writes while it scans.
- **Remove a table or column** → only in a contract step, after grepping the codebase (Server Actions, RPCs, views, other functions, reports) for every reference.

## Every new table: RLS in the same migration

A table in the `public` schema is reachable through the Supabase API the moment it exists. Without RLS enabled, anyone holding the anon key can read and write it — across every tenant. So the migration that creates a tenant table also, in the same file:

```sql
create table public.gastos (
  id uuid primary key default gen_random_uuid(),
  empresa_id uuid not null references public.empresas(id),
  -- ...
);

alter table public.gastos enable row level security;

create policy gastos_select on public.gastos
  for select using (empresa_id = current_empresa_id() and current_tiene_modulo('gastos'));
create policy gastos_insert on public.gastos
  for insert with check (empresa_id = current_empresa_id() and current_tiene_modulo('gastos'));
-- update / delete as needed, each with using + with check

create index gastos_empresa_id_idx on public.gastos (empresa_id);
```

- `empresa_id not null` — a row without a tenant is invisible to everyone or visible to the wrong one, depending on the policy.
- An index on `empresa_id` (usually leading a composite index with the column you filter or sort by) — every policy evaluates it on every query.
- Follow the existing gating pattern of sibling tables. Before adding a role check to a policy, read why the current policies look the way they do (see `multitenant-architecture.md`, the reintroduced-role-check regression).
- Run the security checklist in `multitenant-security.md` against the new table before calling the migration done.

## Locks: don't freeze a table mid-shift

Some DDL takes an exclusive lock; while it waits for or holds that lock, every read and write on the table queues behind it. On a busy table that looks like the whole app hanging.

- **Start migrations with a lock timeout** so a blocked migration fails fast instead of queueing the app behind it:
  ```sql
  set lock_timeout = '5s';
  ```
  If it times out, retry at a quieter moment — don't raise the timeout.
- **Cheap:** adding a nullable column, or one with a constant default (no table rewrite on modern Postgres); creating a new table; creating or replacing a function; adding a policy.
- **Expensive (rewrites or scans the table under lock):** changing a column type, adding a column with a volatile default (`now()`, `gen_random_uuid()` on an existing table), `SET NOT NULL` on a large table, adding a validated foreign key or `CHECK` to a large table.
- **Required column on a big table, without a long lock:**
  ```sql
  alter table public.pedidos add constraint pedidos_canal_not_null
    check (canal is not null) not valid;          -- instant, only checks new rows
  alter table public.pedidos validate constraint pedidos_canal_not_null;  -- scans without blocking writes
  alter table public.pedidos alter column canal set not null;  -- uses the validated check, no full scan
  ```
  The same `NOT VALID` → `VALIDATE` pattern works for foreign keys.
- **Indexes on large tables:** `create index concurrently` avoids blocking writes, but it can't run inside a transaction. If your migration runner wraps each file in a transaction, put it in its own step and run it separately.
- For a small table (a few thousand rows), none of this matters much — say so and keep the migration simple. Check the row count before choosing.

## Backfills and data migrations

- **Idempotent:** safe to run twice (`where nuevo_campo is null`, `on conflict do nothing`). It will get re-run — by you, by a retry, by a reset of a local DB.
- **Batched:** one `UPDATE` over a million rows is one long transaction holding locks and bloating the table. Loop in batches (by id range or per `empresa_id`) and commit between them.
- **Tenant-aware:** derive values within the same `empresa_id`; a join that forgets the tenant column is a cross-tenant data leak written directly into the database.
- **Time zones:** anything that derives a local date or hour follows gotchas #11.
- **Verify before and after:** count the rows you expect to touch, run, count again, spot-check a few tenants.

## Before running a destructive migration

A migration is destructive if it drops a column/table, deletes or overwrites rows, or changes values in place. Before it runs against production:

1. **Confirm a restorable backup exists and how recent it is** (Supabase: the project's backups page or point-in-time recovery; otherwise `supabase db dump` / `pg_dump` of the affected tables right before). "Supabase does backups" is not a plan until you know the restore point and how you'd restore just one table.
2. **Write down the undo.** Supabase migrations are forward-only — there's no automatic down migration. For each destructive step, note in the migration file how to reverse it (or state that it can't be reversed without the backup).
3. **Ask the user explicitly**, naming what will be destroyed. This is a separate authorization from push or merge, like the ones in `SKILL.md`.

## Workflow

1. `supabase migration new <name>` — one concern per migration file; never edit a migration that has already run anywhere shared.
2. Apply locally from scratch (`supabase db reset`) — proves the whole chain still applies in order on an empty database, not just on your hand-tweaked local state.
3. Run the integration suite, including the RLS tests for the tables you touched (`testing-discipline.md`).
4. Check the generated types / RPC callers compile against the new schema.
5. Decide the order explicitly: expand migrations go **before** the deploy that uses them; contract migrations go **after** the deploy that stopped using the old thing. Say out loud which one this is.
6. Apply to production (`supabase db push` or your pipeline), then verify: the app still loads for a real tenant, the RPCs you changed still answer, and the database's security advisor shows nothing new.

## Red flags

- A `RENAME`, `DROP` or type change in the same migration that introduces its replacement.
- `create or replace function` with a different parameter list than the existing function.
- A `create table` in `public` without `enable row level security` in the same file.
- Editing an already-applied migration file instead of writing a new one.
- A backfill that isn't idempotent, isn't batched, or joins without `empresa_id`.
- "It worked on my local DB" without a `supabase db reset` from scratch.
- Dropping anything without having grepped for its references and confirmed a backup.
