# Next.js + Supabase/Postgres gotchas

Concrete bug patterns from this stack, each confirmed in a real production app — not theoretical. Each one cost real debugging time before the mechanism was understood. Read the mechanism, not just the fix — the mechanism is what tells you where else to look.

## 1. A thrown Server Action error is invisible in production

**The mechanism:** Next.js redacts any error that propagates out of a `"use server"` function in production — it replaces the real message with a generic one plus an opaque digest, on purpose (this is documented, intentional behavior, not a bug in Next.js). Locally, in dev mode, you see the real message, so this is invisible until someone hits it against a real deploy.

**What it looks like from the outside:** a feature that clearly has real, specific validation ("no podés reintegrar una venta que nunca se cobró") instead shows a generic, unhelpful error in production, and a user reasonably concludes the whole feature is broken rather than that they hit one specific, well-defined business rule.

**Broken:**
```ts
"use server";
export async function doTheThing(id: string) {
  const { error } = await supabase.rpc("the_thing", { p_id: id });
  if (error) throw error;          // real message dies in prod
}
```

**Fixed — catch it, return it as data:**
```ts
"use server";
export type Result = { ok: true } | { ok: false; error: string };
export async function doTheThing(id: string): Promise<Result> {
  const { error } = await supabase.rpc("the_thing", { p_id: id });
  if (error) return { ok: false, error: error.message };
  return { ok: true };
}
```

The caller then renders `result.error` directly — it's just a string in a normal render path, nothing about it triggers Next's Server Action error handling.

**Where to look for this:** grep every `"use server"` file for `throw` following an `if (error)` check, or for an un-checked `error` that gets returned/ignored silently (the opposite failure — swallowing it so nothing shows at all). If a codebase has fixed this once, audit *every* Server Action file for the same shape — this is exactly the kind of thing that gets copy-pasted as a starting template and then multiplies. In one real audit pass across a ~25-file project, this turned up in **21 of 23 Server Action files, roughly 99 functions** — a single fixed instance (a "no se puede anular" bug report) was the tip of something present almost everywhere else in the codebase, simply never yet reported because nobody had hit those specific validation paths in production yet.

**Two traps found only by auditing exhaustively, not by spot-checking:**
- A function *assumed* to already follow the good pattern (because a sibling function in the same file did) turned out not to — always verify each function individually, don't infer from one neighbor.
- A function with a code comment explicitly discussing this exact redaction problem, sitting right next to a `throw new Error(error.message)` — someone had diagnosed the mechanism correctly and then not actually finished applying the fix (wrapping the real message in `Error` doesn't help; it's still a `throw`, still redacted). A comment describing the problem is not evidence the problem is fixed — check the control flow, not the prose next to it.

## 2. Changing a Postgres function's parameter count creates a silent overload, not a replacement

**The mechanism:** Postgres identifies a function by name *and* parameter type signature. `create or replace function foo(a text, b text)` does not touch an existing `foo(a text)` — it creates a second, distinct function. PostgREST then can't disambiguate a call with the old arity and throws "function is not unique" (`PGRST203`), which surfaces as a confusing RPC failure that has nothing obviously to do with the migration you just wrote.

**Fixed:**
```sql
-- Explicit drop of the OLD signature before creating the new one
drop function if exists public.confirmar_pedido(text, text, boolean);

create function public.confirmar_pedido(
  p_canal text default 'Mostrador',
  p_forma_pago text default 'Efectivo',
  p_pago_pendiente boolean default false,
  p_referencia_pago text default null   -- the new parameter
)
returns uuid ...
```

Applies every time a parameter is added or removed from a function already exposed via RPC — not just when the signature "looks" different, since a single added parameter with a default is exactly the case that's easy to assume `create or replace` handles fine.

## 3. `empresa_id` + RLS is the tenant boundary — not a WHERE clause in application code

Every table that holds tenant data gets an `empresa_id` column and a `for select/insert/update/delete using (empresa_id = current_empresa_id())`-shaped policy (`current_empresa_id()` reading the value off the authenticated session). The application code never filters by tenant manually — it can't, by construction, leak cross-tenant, because the database refuses the query regardless of what the client asked for.

The recurring mistake is *not* forgetting the column — it's writing a query that fetches something correctly-scoped but then treating "all rows this policy returned" as "all rows relevant to what I'm about to render," which is a different, UI-level bug (see next item) rather than a security one. Verify which one you're looking at before assuming a security hole: check whether the policy itself is missing the `empresa_id` predicate (real leak) versus whether the query is tenant-safe but returning a broader set than the UI should display (UX gap, item 4 below).

## 4. A filter populated from the full lookup table, not from values actually present

**The mechanism:** a filter/pill/dropdown built as `theWholeCatalog.map(item => <option>...)` looks correct and reads correct in review — every valid value is technically an option. But a "filter" is supposed to narrow an already-rendered list, and any catalog entry with zero matching rows in that list is a dead end: clicking it always produces "no results," and the user has no way to tell which pills are real without clicking each one.

**Broken:**
```tsx
<FiltrosCategoria categorias={categorias} .../>  {/* every category the tenant ever created */}
```

**Fixed — derive the option list from what's actually being filtered:**
```tsx
const categoriasEnUso = categorias.filter(c => productos.some(p => p.categoria_id === c.id));
<FiltrosCategoria categorias={categoriasEnUso} .../>
```

**The distinction that matters:** this fix applies to *filter/narrow* contexts only. An *assignment* dropdown (choosing a category while creating/editing a record) should absolutely show the full catalog, including entries with nothing assigned yet — that's not a bug, it's the only way to ever assign the first item to an empty category. Don't "fix" those; check which kind of control you're looking at before touching it. When one instance of this bug gets fixed, the highest-value follow-up is grepping every other filter/pill-selector in the app for the same "maps the raw catalog with no `.filter()` against the actual data" shape — this tends to repeat across every screen that was built from the same original template, since a filter and an assignment dropdown often look identical in the code until you check what they're feeding. One real audit pass found the exact same shape twice more in a second, structurally unrelated part of the app (a "filter contracts by tenant" and a "filter expenses by property" control, both built from the same original filter-bar template as the one already fixed) — confirming this really does propagate across a codebase via copy-paste, not just in theory. A hardcoded, small, fixed-enum filter (three or four literal string options, not a growable catalog table) carries the same theoretical risk but is much lower priority — the failure mode only bites once real data creates a channel/status that's never actually used, which is rarer and lower-stakes than a user-editable catalog.

## 5. Hand-rolled bar/progress visuals: percentage height needs a *direct* parent with resolved height

**The mechanism:** CSS percentage-height resolution is checked against the immediate parent's own computed height, not any ancestor further up. A grandparent with `h-40` does nothing for a percentage set on a grandchild if the child in between is sized by content (`height: auto`, e.g. plain `flex flex-col` with no explicit height and no `flex-1`/stretch pulling a real size down to it). The element silently renders at `height: 0` — no console error, no layout warning, it just doesn't show up.

**Broken:**
```tsx
<div className="flex h-40 items-end gap-2">
  <div className="flex flex-1 flex-col items-center justify-end">  {/* height: auto */}
    <div style={{ height: `${pct}%` }} className="bg-blue-500" />   {/* resolves against auto → 0 */}
  </div>
</div>
```

**Fixed — an intermediate wrapper that actually inherits a resolved size:**
```tsx
<div className="flex h-40 items-end gap-2">
  <div className="flex h-full flex-1 flex-col items-center gap-1">      {/* h-full: resolves against h-40 */}
    <span>{label}</span>
    <div className="flex w-full flex-1 items-end">                     {/* flex-1 in a sized flex column: resolved */}
      <div style={{ height: `${Math.max(pct, 2)}%` }} className="bg-blue-500" />
    </div>
  </div>
</div>
```

Note the `Math.max(pct, 2)` — a real zero-or-near-zero data point should still render a sliver, not vanish, or it reads as the same bug even once the mechanism is fixed.

**Width percentages on block-level elements mostly don't have this problem** — a block box's `width: auto` resolves to fill its containing block by default (that's normal-flow behavior), where `height: auto` shrink-wraps to content. A `<div style={{width: pct+'%'}}>` inside a plain (non-flex, non-inline-block) block-level track div is very likely fine without an intermediate wrapper; verify by checking whether anything in the ancestor chain uses `inline-block`/flex with a size determined by content, but don't assume every percentage-styled element needs the same treatment as the height case.

## 6. `npm install` warnings are not build failures

`npm warn deprecated ...` and `npm warn allow-scripts ...` (unapproved postinstall scripts) show up on every install of a project with a handful of common transitive dependencies, succeed or fail identically regardless of these warnings, and tell you nothing about whether the actual build/deploy worked. When troubleshooting "did my deploy work," skip past these lines to the part of the log that says `Compiled successfully`, lists the actual routes, and ends in `Build Completed` / `Deployment completed` (or their equivalents on the target platform) — and check the deployment's commit hash against the merge commit you expect, since that's the one piece of log output that actually answers the question.

## 7. Infra gotchas specific to a local Supabase + Docker dev loop

- **Stale Kong routing after `supabase db reset`:** the local API gateway container can end up pointing at a stale upstream IP after a reset, producing opaque `502`/connection errors that look like the reset itself failed. Fix: `docker restart <kong-container-name>`, wait a few seconds, verify `curl http://127.0.0.1:54321/auth/v1/health` returns 200 before retrying whatever failed.
- **Never run a production build and a dev server in the same directory at the same time.** They share a `.next/` build-cache directory and will corrupt each other's output — the failure mode is bizarre (a page renders as raw unstyled HTML, or a stale dev bundle serves after a supposedly-clean build) and looks exactly like a real application bug. If you need both, use separate worktrees/checkouts, or stop one before running the other.
- **Docker Desktop can silently stop during a long idle period**, and every test in the suite fails at once. A sudden 100%-failure run after a long gap is a Docker-down signal before it's a real-regression signal — check `docker ps` first.

## 8. `position: sticky` inside a sidebar-layout content pane needs a *height-bounded* scroll container, not just `overflow-y-auto`

**The mechanism:** a common layout for "fixed sidebar + independently-scrolling content" is `flex` row with the content pane getting `flex-1 overflow-y-auto`. That alone is not enough. If the row's own height is only `min-h-screen` (a floor, not a ceiling) instead of `h-screen` (fixed), the content pane's height stretches to fit whatever it contains instead of being capped at the viewport — so it never actually overflows *itself*, and the real scrolling happens on the document/`html` element instead. The pane still has `overflow-y: auto` declared, and per spec that alone is enough to make it the "nearest scrolling ancestor" that any descendant `position: sticky` element positions against — **even though this ancestor never scrolls in practice.** The browser is not lying: `getComputedStyle(el).position` genuinely returns `"sticky"`. But since the scroll container it's pinned against never moves independently of the document, the element just travels 1:1 with the real page scroll — visually indistinguishable from `position: static`.

**What it looks like from the outside:** a feature that was implemented correctly, code-reviewed, and "verified" (CSS class present, computed style checked) still gets reported by a real user as "doesn't float, stays at the top" — and every earlier verification pass looked fine because it only ever asked the DOM "what does this compute to," never "does this visually track the scroll the way it should."

**Broken:**
```tsx
<div className="flex min-h-screen ...">      {/* no ceiling — grows to fit content */}
  <Sidebar />
  <div className="flex-1 overflow-y-auto ...">
    <SomeCard className="sticky top-6 ..." />   {/* getComputedStyle says sticky; never actually sticks */}
  </div>
</div>
```

**Fixed — bound the row to the viewport so the inner pane actually has to scroll:**
```tsx
<div className="flex h-screen overflow-hidden ...">   {/* fixed ceiling */}
  <Sidebar />
  <div className="flex-1 overflow-y-auto ...">          {/* now genuinely scrolls when content overflows */}
    <SomeCard className="sticky top-6 ..." />           {/* now really sticks */}
  </div>
</div>
```

**The verification lesson this exposes:** "verify by seeing it, not by inferring it" (see SKILL.md) has a sharp edge here — checking `getComputedStyle(el).position === 'sticky'` feels like seeing it, but it only confirms the CSS declaration parsed and applied, not that the *effect* the declaration is supposed to produce actually happens. The only real check is behavioral: change the scroll position (real user scroll, or `scrollContainer.scrollTop = N` in a script) and read the element's `getBoundingClientRect()` before and after. If the top coordinate keeps changing 1:1 with the scroll delta instead of clamping at the offset you set, it isn't actually sticking, no matter what the computed style says. This also means: when a user reports "the thing you fixed still doesn't work" on something you already "verified," don't reflexively assume the user is looking at a stale deploy or the wrong viewport — check whether your own verification actually exercised the effect, or just its precondition.
