---
name: the-seedskill
description: Use when building, fixing or evolving a production web app that already has real users, especially a multi-tenant SaaS (reference stack Next.js + Supabase/Postgres + Tailwind + Vercel). Use when a bug report, failing test or "it looks completely broken" report arrives; before proposing a fix; before claiming something is fixed, passing or done; before writing code for a new feature or behavior change; when writing or changing tests; when receiving or requesting code review; when deciding whether to push, merge or clean up a branch; when writing or applying a database migration; when adding a table, RPC, Server Action or permission, or reviewing security; when doing a responsive/mobile pass; when designing one codebase to serve several distinct businesses/tenants; and when asked to turn a project's lessons into a reusable method. Push harder to consult this on push/merge authorization, migrations and Postgres RPC/RLS patterns, tenant isolation, and "broken in production but tests are green."
---

# The Seedskill

## Why this exists

This captures the operating discipline that came out of actually shipping a multi-tenant restaurant/property-management SaaS solo, over many rounds of: build → user tries it for real → something's subtly wrong → find the *actual* mechanism → fix it → notice the same mechanism is probably lurking elsewhere → check. It pairs a compact core of general engineering discipline (debugging, verification, testing, design gates, review) with the specific residue: the gotchas that cost real debugging time, the design calls that turned out right, and the calls that had to be reversed after the user actually used the thing.

**Read `references/` files when the situation matches — don't preload them.**

| When | Read |
|---|---|
| A bug, failing test or "looks broken" report arrives — and again right before you claim anything is fixed or done | `references/debugging-and-verification.md` — four-phase root-cause process, the three-fix rule, the verification gate |
| Writing or changing a test, or fixing any bug | `references/testing-discipline.md` — red→green→refactor, tests that name the break, multi-tenant tests worth writing every time |
| A request to build or change behavior (not a reported bug) | `references/design-plan-execute.md` — classify spike/bounded/architectural, approval gates, writing a plan, finishing a branch |
| Review feedback arrives, a branch is about to be integrated, or several independent problems need work at once | `references/review-and-agents.md` — receiving/requesting review, parallel agents |
| Writing or applying a migration: new table/column, changed RPC signature, RLS policy, index, backfill, anything destructive | `references/safe-migrations.md` — expand→migrate→contract, RLS in the same migration, lock-safe DDL, backups before destructive changes |
| Shipping a new table, RPC, Server Action, route, bucket or permission change — or asked for a security review | `references/multitenant-security.md` — the tenant-isolation checklist: what bypasses RLS, public endpoints, cross-tenant tests |
| "Works locally, broken in prod", writing a Server Action, touching a Postgres function exposed via RPC | `references/nextjs-supabase-gotchas.md` — concrete bug patterns in this stack, each with a broken/fixed pair |
| Starting a new SaaS, or adding a second vertical/tenant-type to an existing one | `references/multitenant-architecture.md` — one codebase for structurally different businesses without forking |
| A responsive/mobile pass, or any UI change visual enough to deserve a mockup first | `references/responsive-design-method.md` — breakpoint strategy and mockup-first process |

The first four are the general engineering discipline, adapted from [obra/superpowers](https://github.com/obra/superpowers) (MIT) so this skill stands on its own. The rest are the domain-specific layer for this stack.

## The rules that don't bend

1. **No fix without a root cause.** Reproduce and name the mechanism before touching code.
2. **No "done" without fresh evidence.** Run the check in this turn, read the output, then claim — and for anything visual, look at the screen.
3. **Logic gets its test first, and you watch it fail.** Every bug fix starts with a test that reproduces it.
4. **No implementation before the design gate.** Classify the request, show the design at the size it deserves, wait for a yes.
5. **Push and merge are separate, per-instance authorizations.**
6. **The database is the tenant boundary.** Every new table ships with RLS in the same migration; `empresa_id` comes from the session, never from input.
7. **Schema changes never break the running version.** Expand → migrate → contract; destructive steps only with a confirmed backup and an explicit yes.
8. **After every fix, audit for the same shape** across the codebase.

User instructions (CLAUDE.md, direct requests) take precedence over this skill; skip one of these rules only when the user has explicitly said to.

## If you also have superpowers installed

This skill is self-contained, but if the [superpowers](https://github.com/obra/superpowers) plugin is installed, its skills cover the general process in more depth. Use them for the *how*, and this skill for the stricter rules and the stack-specific layer:

| Situation | superpowers skill | What this skill adds or tightens |
|---|---|---|
| Starting a feature | `brainstorming`, `writing-plans` | Mockup-first for anything visual; the multi-tenant "review focus" inputs (empty tenant, lower role, phone viewport) |
| Isolating work | `using-git-worktrees` | Never run a production build in the directory a dev server is using |
| A bug report comes in | `systematic-debugging` | Where root causes actually hide in this stack (gotchas reference), so phase 1 has somewhere concrete to look |
| Claiming a fix works | `verification-before-completion` | For UI, green tests are necessary but not sufficient — see "Verify by seeing it" |
| Wrapping up a branch | `finishing-a-development-branch` | Push and merge are separate, per-instance authorizations |

## Push and merge are separate, per-instance authorizations

In a real client-facing app, "push it" does not imply "merge it," and approving one push does not carry over to the next one — even minutes later, even for the same kind of change. Ask again, every time, for both push and merge specifically. This feels redundant until the one time it isn't: a merge is visible to everyone and hard to cleanly undo once deployed, a push to a feature branch is not. Treat them as different weight classes, not two steps of one approval.

If a merge is blocked by a permissions/policy layer you don't control (a CI gate, an org policy, a classifier), don't route around it by finding another tool that accomplishes the same effect (e.g. merging locally and pushing the result when the direct merge command was refused). Explain the block and hand the decision back.

After a merge: sync the base branch locally (fast-forward only), then clean up the worktree and branch (local + remote) — but only once you've confirmed the merge actually happened, and only if the user hasn't indicated they want to keep iterating on that branch.

## Verify by seeing it, not by inferring it

A green build and a green test suite prove the code compiles and the logic you thought to test does what you thought. Neither proves a human looking at the screen sees what you intended. For anything with a visual or interactive surface:

1. Actually load the page in a real browser session, logged in as a real (seeded) user with a real role.
2. Reproduce the *exact* scenario in the bug report before touching code — same screen, same data shape, same viewport if it's layout-related. "It works at 1280px" and "it works" are different claims.
3. After the fix, re-run that exact scenario and look at it, not just at the diff.
4. Scale the rest of verification (unit/integration suite, full e2e) to the blast radius of the change, not to habit: a pure CSS/copy change doesn't need the full e2e cycle every time; a change to a shared component's props, a Server Action's return shape, or anything touching money/permissions does. Say out loud which tier you're running and why, so it's a visible decision rather than a default.
5. Never run a production build (`next build` / equivalent) in the same directory a dev server is currently running in — they fight over the same build cache and the corruption looks exactly like a real bug (missing CSS, raw unstyled HTML) until you realize it's your own tooling. Stop the dev server first.

## Diagnosing a "this is broken" report

Real reports from a real user testing a real feature are gold — and also frequently underspecified. Before writing any fix:

- **Reproduce it yourself, in the same conditions, before forming a theory.** "I can't see X" often decomposes into several genuinely different bugs that happen to produce similar-looking symptoms (a table with invisible columns and a chart with invisible bars are not the same root cause just because both look "not there").
- **Check whether it's actually a bug, or working-as-designed-but-not-as-liked.** Software can be functioning exactly as built and still be wrong for the user — most often at a breakpoint or condition boundary they didn't realize they were testing at (a sticky panel that's correctly *not* sticky in the 200px band it was deliberately scoped to, tested from a half-width browser window). Confirm the computed behavior first with a quick script/console check before assuming the code is wrong.
- **When two interpretations of a request lead to opposite implementations, ask one direct question before building either one.** Don't spend a build-test-deploy cycle guessing. But once you've asked and they answer with something garbled or self-contradictory (typos, dropped words — very common from a phone), restate your best-effort interpretation in one sentence and let a bare "sí"/"yes" confirm it, rather than asking them to re-explain from scratch a second time.
- **A request to undo something the user explicitly asked for earlier is normal, not a failure signal.** Features that sound right in the abstract sometimes feel wrong once you're actually scrolling past them fifty times a shift. Implement the reversal as cleanly as you built the original — no need to relitigate whether the original ask was "correct."

## After every fix, audit for the same shape

The single highest-leverage habit in this whole playbook: a bug is rarely alone. It's a *pattern* that got typed once and then copy-pasted, or a mechanism (a framework behavior, a CSS resolution rule, a permission check) that applies uniformly to every place someone used the same construct without knowing about the edge case. When you fix one instance:

1. Name the actual mechanism, not just the symptom ("percentage height doesn't resolve against an auto-sized parent" — not "the chart was broken").
2. Grep/search the rest of the codebase for the same construct (same library-less bar-chart pattern, same `"use server"` + un-caught `throw`, same "show every row from the lookup table as a filter option" shape).
3. Report findings as an inventory before fixing anything else — a bug class found via systematic search is worth far more to the user than the one instance they happened to notice, and finding it *before* they hit it in production is the whole point.

This applies just as much to your own conventions as to bugs: if you establish a good pattern once (a typed `{ok, error}` return instead of a thrown exception; an explicit "Guardar cambios" button instead of a silent autosave), and later add a new feature that doesn't follow it, that's the same class of drift — catch it the same way.

Don't undersell how far this goes: on one real project, fixing a single "can't anull a sale" bug report (root cause: a Server Action threw a raw error that Next.js redacted in production — see gotchas reference #1) prompted an exhaustive audit of every Server Action in the codebase, which turned up the same raw-throw shape in **21 of 23 files, on the order of 99 functions** — one fixed bug report was standing in for a latent class present almost everywhere else, just not yet triggered. The same audit pass, applied to a different fixed bug (an unfiltered catalog used as a filter — gotchas reference #4), found two more live instances of that exact shape elsewhere in the app within minutes of looking. Neither of those would have been found by waiting for a user to report them one at a time. A systematic grep after the first fix is cheap; finding out about instance #47 from an angry user months later is not.

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "They already said push earlier, this is basically the same" | Ask again. Push and merge are separate authorizations, and so is every subsequent push. |
| "Tests are green, feature's done" | Green tests prove the tests you wrote pass. Load the actual screen before calling it done. |
| "It's just responsive/CSS, e2e would be overkill" — *and also* "better run the full suite just in case, every time" | Neither default is right — say which tier you're running and why, matched to what actually changed. |
| "The user's wording is contradictory, I'll just pick one" | Restate your best interpretation in one line and let them confirm — cheaper than a wrong deploy. |
| "They asked for sticky, now they don't want it — I must have built it wrong" | Read it again: they're describing normal iteration, not a bug in your implementation. Ship the reversal. |
| "I found one instance, that's the fix" | Grep for the same shape elsewhere before moving on — that's where most of the value is. |
| "It compiled and the page loaded, good enough" | A blank chart or an invisible filter also "loads." Look at the actual pixels for anything visual. |
| "The fix is obvious, I'll skip the investigation" | Seeing the symptom isn't understanding the mechanism. Reproduce and trace first — it's faster than thrashing. |
| "One more fix attempt" (after two failed ones) | Three failed fixes means the architecture is wrong, not the hypothesis. Stop and discuss the design. |
| "I'll write the test after it works" | A test you never saw fail proves nothing. Write it first, watch it fail. |
| "It's too small to need a design" | Small means a two-sentence design in chat — then wait for the yes. The gate is the approval, not the length. |
| "The agent/reviewer said it's done/right" | Read the diff yourself; verify the suggestion against the codebase before acting on it. |
| "It's just a rename / one small column change, one migration is fine" | Open tabs still run the old code. Expand → migrate → contract, even for a rename. |
| "RLS protects it" (about a view, a `security definer` RPC, or a service-role job) | Those bypass RLS. Check the tenant inside, or it's open to every tenant. |
