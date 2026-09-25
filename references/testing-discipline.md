# Testing discipline

Adapted from the `test-driven-development` skill and its `writing-good-tests` reference in [obra/superpowers](https://github.com/obra/superpowers) (MIT). Read this before writing or changing a test, and whenever you fix a bug.

## How this fits with "scale verification to blast radius"

`SKILL.md` says to match the *breadth* of verification (unit only, integration, full e2e, visual check) to what changed. That's about which suites you run. It does not relax the rule below, which is about how each piece of logic gets its test: **for logic you write or fix, the test comes first and you watch it fail.** A pure CSS/copy change has no logic to test-drive; a Server Action, RPC, permission check or money calculation always does.

## Red → green → refactor

1. **Red.** Write one minimal test for one behavior, with a name that says what should happen. Use real code; mock only what's slow or external.
2. **Verify red — mandatory.** Run it. It must *fail* (not error), for the expected reason (the behavior is missing, not a typo or import error). If it passes immediately, you're testing behavior that already exists — fix the test.
3. **Green.** Write the simplest code that passes. No extra options, no "while I'm here". YAGNI.
4. **Verify green — mandatory.** The new test passes, **the project's whole suite** passes, output is clean. A green run of your one file is not a green suite. Report any failure you see by name, including ones you didn't cause.
5. **Refactor** only while green: remove duplication, improve names, extract helpers. Don't add behavior.

If you wrote implementation code before its test, the honest options are to delete it and restart test-first, or to tell the user explicitly that this piece was tested after the fact. Don't quietly bolt tests onto it and call it TDD — a test you never saw fail hasn't proven it can catch anything.

**Bug fixes always start with a failing test that reproduces the bug.** For a regression test, prove it: revert the fix → test fails; restore → test passes.

## Writing tests that actually catch things

**Name the break.** Before writing the body, answer: *what production change would make this test fail — and is that change a bug or a decision?* Can't name one → redesign the test around observable behavior.

**Derive expectations independently.** Use literals and hand-checked fixtures. An expected value computed by the code under test (or its helpers) passes no matter what:

```ts
// ❌ Mirror assertion — always true
const expected = calcTotal(order);
expect(calcTotal(order)).toBe(expected);

// ✅ Hand-derived literal
expect(calcTotal({ items: [{ price: 1200, qty: 2 }], tip: 300 })).toBe(2700);
```

**No change detectors.** A test that only fails when you intentionally change a constant or a message wording fires on redesign and sleeps through bugs. Not `expect(MAX_RETRIES).toBe(5)` — test "a failing call is retried 5 times and the 6th never happens."

**Your code, not the framework.** Test the contract at your boundaries — the payload your Server Action returns (`{ok, error}`), the rows your RPC writes, what a user of role X can and can't see. Don't test that Next.js calls your handler or that Supabase executes SQL.

**Assert real behavior, never mock behavior.** If the assertion is about what the mock did, you've tested the mock. Understand a dependency's side effects before mocking it; keep test-only helpers in test utilities, never in production modules.

**Multi-tenant specifics worth a test every time:**
- Tenant A cannot read or write tenant B's rows (RLS), from each role.
- A lower role can't reach an action the UI merely hides from it.
- The error path of every Server Action returns a typed error instead of throwing (see gotchas #1).

## When testing is hard

| Problem | What it's telling you |
|---|---|
| Don't know how to test it | Write the API you wish existed, then the assertion first. |
| Test is complicated | The design is complicated. Simplify the interface. |
| Must mock everything | Code is too coupled. Inject dependencies. |
| Setup is huge | Extract helpers; if it's still huge, simplify the design. |

## Rationalizations

| Excuse | Reality |
|---|---|
| "Too simple to test" | Simple code breaks; the test takes 30 seconds. |
| "I'll test after" | Tests written after pass immediately — which proves nothing. |
| "Already tested it manually" | Manual checks leave no record and can't be re-run on the next change. |
| "Deleting this work is wasteful" | Sunk cost. Keeping code you can't trust is the waste. |
| "Existing code has no tests" | You're touching it — add tests for the part you touch. |
