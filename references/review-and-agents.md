# Code review and working with agents

Adapted from the `receiving-code-review`, `requesting-code-review`, `dispatching-parallel-agents` and `subagent-driven-development` skills in [obra/superpowers](https://github.com/obra/superpowers) (MIT). Read this when review feedback arrives (from the user, a teammate or a bot), before integrating a branch, or when there are several independent problems to work on at once.

## Receiving feedback

Technical correctness over social comfort. The response pattern:

1. **Read** all of it before reacting.
2. **Restate** each item in your own words — or ask.
3. **Verify** it against the actual codebase.
4. **Evaluate**: is it right for *this* codebase, stack and tenant model?
5. **Respond** with a technical acknowledgment or reasoned pushback.
6. **Implement** one item at a time, testing each.

**If any item is unclear, clarify before implementing any of them.** Items are often related; implementing 1, 2, 3 and 6 while guessing at 4 and 5 produces a half-right change. Say "I understand 1, 2, 3, 6 — need clarification on 4 and 5."

**No performative agreement.** Skip "You're absolutely right!", "Great catch!", "Thanks for…". State the fix ("Fixed — the action now returns `{ok:false}` instead of throwing, in `sales/actions.ts`") or just show it in the code.

**From the user:** trusted — implement once you understand the scope. **From external reviewers or bots:** check before implementing — does it break something, is there a reason for the current code, does the reviewer have the full context? If it conflicts with a decision the user already made, stop and ask the user.

**YAGNI check.** When a reviewer asks to "implement it properly" (metrics, export, pagination, config options), grep for actual usage first. Nothing calls it → propose removing it instead.

**Order:** clarify everything unclear → blocking issues (breakage, security, tenant leaks) → simple fixes → complex fixes → verify no regressions.

**Push back when** the suggestion breaks existing behavior, lacks context, violates YAGNI, is wrong for this stack, or contradicts the user's architecture. Use evidence (code, tests, docs), not defensiveness. If you pushed back and were wrong, say so in one line and fix it.

## Requesting review

- Mandatory before merging to the main branch; worth it after a complex bug fix or when stuck.
- The reviewer (a subagent, a review tool, or a human) gets **precisely crafted context**: what was built, the requirements or plan it should satisfy, and the commit range (`git merge-base origin/main HEAD`..`HEAD`). Never hand over your whole session — it anchors the reviewer on your reasoning instead of the work.
- Ask for findings by severity. Critical → fix now. Important → fix before proceeding. Minor → note it. Wrong → push back with reasoning.
- A useful two-stage review for plan-driven work: first *spec compliance* (does it do what was asked, nothing more, nothing less?), then *code quality*.

## Parallel agents

Use when there are 2+ problems that are genuinely independent — different failing test files, different subsystems, different bugs — with no shared state.

Don't use when failures might be related (fixing one may fix the others), when you need the whole-system picture, or when agents would edit the same files.

Each agent gets:
- a narrow scope (one test file, one subsystem),
- a clear goal ("make these tests pass", "find the root cause of X"),
- constraints ("don't change code outside `app/(pos)/`"),
- an expected output (summary of root cause and changes).

Dispatch them together so they actually run concurrently. When they return: read each summary, check the changes don't conflict, run the full suite, and **verify each agent's claims against the diff** — an agent reporting "success" is not evidence (see the verification gate in `debugging-and-verification.md`).
