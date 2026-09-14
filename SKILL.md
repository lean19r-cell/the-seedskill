---
name: multitenant-saas-playbook
description: Engineering playbook for building and evolving a real, paying-customer-facing multi-tenant SaaS (Next.js + Supabase/Postgres + Tailwind + Vercel is the reference stack, but the lifecycle and diagnostic habits generalize). Use whenever shipping a feature or fix to a production web app that already has real users, whenever a user reports a UI bug that "looks completely broken," whenever doing a responsive/mobile pass on an existing app, whenever designing a system meant to serve multiple distinct businesses/tenants from one codebase, and whenever the user asks to turn accumulated lessons from a project into a reusable method. Push harder to consult this on anything touching git worktree lifecycle + push/merge authorization, Postgres RPC/RLS patterns, or "why does this look broken in production but tests are green."
---

# Multi-tenant SaaS playbook

## Why this exists

This captures the operating discipline that came out of actually shipping a multi-tenant restaurant/property-management SaaS solo, over many rounds of: build → user tries it for real → something's subtly wrong → find the *actual* mechanism → fix it → notice the same mechanism is probably lurking elsewhere → check. It is not a generic "how to code" guide — the generic pieces already live in other skills (see below). This is the specific residue: the gotchas that cost real debugging time, the design calls that turned out right, and the calls that had to be reversed after the user actually used the thing.

**Read `references/` files when the situation matches — don't preload them.**
- `references/nextjs-supabase-gotchas.md` — concrete bug patterns in this stack, each with a broken/fixed code pair. Read this whenever you're debugging a "works locally, broken in prod" report, writing a Server Action, or touching a Postgres function that's already exposed via RPC.
- `references/multitenant-architecture.md` — how to design one codebase that serves structurally different businesses (a restaurant POS and a property-rental manager, say) without forking. Read this when starting a new SaaS or when asked to add a second "vertical"/tenant-type to an existing one.
- `references/responsive-design-method.md` — the breakpoint strategy and mockup-first process that actually survived contact with a real user testing on a real phone. Read this before a responsive/mobile pass, or when a UI change is visual enough that a mockup should come before code.

## How this composes with other skills

This playbook assumes you're already applying the general-purpose skills — it adds the domain-specific layer on top, and tightens a couple of things past their defaults:

| Situation | Base skill | What this playbook adds |
|---|---|---|
| Isolating work | `using-git-worktrees` | Nothing extra on *how* to create one — but see "Push and merge are separate, per-instance authorizations" below |
| A bug report comes in | `systematic-debugging` | The stack-specific places root causes actually hide (see gotchas reference) — so Phase 1 evidence-gathering has somewhere concrete to look first |
| Claiming a fix works | `verification-before-completion` | For a UI feature, green tests are necessary but not sufficient — see "Verify by seeing it" below |
| Wrapping up a branch | `finishing-a-development-branch` | A stricter authorization rule for this kind of app — see below |

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
