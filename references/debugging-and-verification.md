# Debugging and verification

Adapted from the `systematic-debugging` and `verification-before-completion` skills in [obra/superpowers](https://github.com/obra/superpowers) (MIT), trimmed and merged with what this playbook learned in production. Read this when a bug report, test failure or "looks broken" report comes in, and again right before you claim anything is fixed or done.

## Two iron laws

```
NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
```

Both matter most exactly when they feel least necessary: under time pressure, when the fix "is obvious", when you've already tried something and it didn't work.

## The four phases

Finish each phase before starting the next.

### 1. Root cause investigation

1. **Read the error completely.** Stack trace, line numbers, error codes, warnings you'd normally scroll past. In Next.js production builds, remember the message may be redacted (see `nextjs-supabase-gotchas.md` #1) — get the real one from server logs before theorising.
2. **Reproduce it reliably**, in the same conditions as the report (same role, same data shape, same viewport). Not reproducible → gather more data, don't guess.
3. **Check what changed.** `git log`, `git diff`, new deps, env vars, migrations, config.
4. **In multi-component systems, instrument every boundary before fixing.** Browser → Server Action → RPC → Postgres function → RLS policy: log what enters and what exits each layer, run once, and let the evidence tell you *which* layer breaks. Then investigate only that layer.
5. **Trace the bad value backwards** to where it originates. Fix at the source, not where it surfaced.

### 2. Pattern analysis

- Find similar code in the same repo that **works**. List every difference between working and broken, however small — don't pre-filter with "that can't matter."
- If you're following a reference implementation or doc, read it completely. Skimming a pattern and "adapting" it is how half of the gotchas in this playbook were born.

### 3. Hypothesis and minimal test

- Write it down: *"I think X is the root cause because Y."*
- Change **one** variable to test it. Worked → phase 4. Didn't → new hypothesis; don't stack a second fix on top of the first.
- If you don't understand something, say so. "I don't know why X" is a valid, useful status.

### 4. Implementation

1. Write a failing test (or a one-off script) that reproduces the bug — see `testing-discipline.md`.
2. Make a single fix for the root cause. No "while I'm here" refactors bundled in.
3. Verify: the new test passes, the rest of the suite passes, **and** the original user-visible scenario is gone (look at it — see "Verify by seeing it" in `SKILL.md`).
4. Then do the audit-for-the-same-shape pass from `SKILL.md`.

### The three-fix rule

If a fix doesn't work, count. Under three attempts → back to phase 1 with the new information. **Three or more failed fixes means the problem is architectural, not a bad hypothesis** — typical signs: each fix reveals new coupling somewhere else, fixes need "massive refactoring", every fix creates a new symptom. Stop and discuss the design with the user before attempt #4.

### When there truly is no root cause

Sometimes it really is environmental or timing-dependent. Then: document what you investigated, add appropriate handling (retry, timeout, clear error message), add logging so the next occurrence carries evidence. But treat this as the rare case — most "no root cause" conclusions are an investigation that stopped early.

## Hardening after the fix

**Defense in depth.** When the bug came from invalid data, one validation check can be bypassed by another code path later. Validate at each layer the data crosses so the bug becomes structurally impossible:

| Layer | In this stack |
|---|---|
| Entry | Zod/typed parsing at the Server Action boundary |
| Business logic | Guards inside the action / RPC before the write |
| Database | `CHECK` constraints, `NOT NULL`, FKs, RLS policies |
| Forensics | Structured logs with tenant id + user id on the failure path |

**Condition-based waiting.** Flaky tests usually guess timing with `sleep`/`setTimeout`. Wait for the actual condition instead (`await expect(locator).toBeVisible()`, poll for the row to exist, wait for the event), always with a timeout and a descriptive error. An arbitrary delay is only acceptable when you're testing timing itself (debounce, throttle) — and then comment why.

## The verification gate

Before any status claim — "fixed", "passing", "done", "works now", or even "Great!" — run this:

1. **Identify** the command or observation that proves the claim.
2. **Run** it fresh and complete, in this turn.
3. **Read** the full output: exit code, failure count, warnings.
4. **Compare**: does the output actually confirm the claim? If not, report the real status with the evidence.
5. Only then make the claim, and show the evidence alongside it.

| Claim | Requires | Not sufficient |
|---|---|---|
| Tests pass | Test run output with 0 failures | A previous run, "should pass" |
| Build succeeds | Build command exit 0 | Lint passing, dev server "looks fine" |
| Bug fixed | Original symptom reproduced and now gone | Code changed, assumed fixed |
| Regression test works | Red → green verified (revert fix, test fails; restore, test passes) | Test passes once |
| UI works | You loaded the page and looked at it | Green tests + clean diff |
| Subagent finished | You read the diff yourself | The agent said "success" |
| Requirements met | Line-by-line check against the request | Tests passing |

## Red flags — stop and go back

- "Quick fix for now, investigate later."
- "Let me just try changing X and see."
- Proposing fixes before you can name the mechanism.
- "One more attempt" after two failed ones.
- Words like *should*, *probably*, *seems to* in a status report.
- About to commit, push or open a PR without having run the checks in this turn.
- The user says "stop guessing", "is that actually happening?", or "are we stuck?" — that's a signal you skipped phase 1.
