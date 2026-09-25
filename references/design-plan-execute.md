# Design → plan → execute

Adapted from the `brainstorming`, `writing-plans`, `executing-plans` and `finishing-a-development-branch` skills in [obra/superpowers](https://github.com/obra/superpowers) (MIT), fitted to a solo builder shipping to real paying users. Read this when a request is about building or changing behavior (not just fixing a reported bug), before writing code.

## 1. Classify the request out loud

Say which path you're on so the user can override it ("this looks bounded, so I'll give you a short design here instead of a spec"):

| Path | What it is | Artifact | Gate before code |
|---|---|---|---|
| **Spike** | "Can we…?", "is it possible…?" — the output is an answer, not code you keep | 2–3 sentences: the question and what you'll try | A nod |
| **Bounded** | A well-scoped change to a flow that already exists in this repo (a new field, a filter, a one-screen fix) | Short design in chat: approach, files touched, how you'll test | Explicit "yes" |
| **Architectural** | New project, new vertical/tenant-type, new subsystem, schema or interface changes others depend on | Written spec, then a written plan | Spec approved, then plan approved |

Rules:
- When in doubt between two paths, take the heavier one. Hidden complexity discovered mid-task upgrades the path — stop and say so. Nothing downgrades mid-task.
- "Bounded" measures the repo, not your familiarity with the kind of app. No existing flow to read → not bounded.
- A spike's code stays throwaway. Keeping it is a new request that gets its own classification.
- Each approval covers the stage you actually showed. Approving an idea doesn't approve a spec that doesn't exist yet.
- Read-only exploration is always allowed before the gate.

## 2. Understand intent before proposing

- Find out *why*: the outcome, who it's for, what success looks like. If that's missing, ask **one** focused question at a time — preferably multiple choice.
- Write back your understanding in a few lines, separating what they said from what you're assuming, and let them correct it. If the request already states purpose and constraints, reflect it instead of re-asking.
- For architectural work, propose 2–3 approaches with trade-offs and a recommendation, then present the design in sections small enough to actually read, confirming each.
- For anything visual, the mockup-first process in `responsive-design-method.md` applies: show the picture before writing code.
- Ruthlessly apply YAGNI: cut features the stated goal doesn't need.

## 3. Write the plan (architectural path)

Write it for a capable engineer with zero context on this codebase:

- **Header:** goal (one sentence), architecture (2–3 sentences), stack, link to the spec.
- **Global constraints:** project-wide rules copied verbatim from the spec (tenant isolation, roles, currency/locale, copy rules).
- **Review focus:** the ~5 inputs or conditions the spec doesn't mention but a real user will hit (empty states, a tenant with zero rows, a role with no permission, a phone viewport, a slow network). Each one gets a test in the task that owns it.
- **File map first:** which files are created or modified and what each is responsible for. One responsibility per file; follow the repo's existing patterns.
- **Tasks:** each is the smallest unit that carries its own test cycle and can be reviewed on its own. Each lists exact file paths, the interfaces it consumes and produces, and steps of 2–5 minutes: write failing test → run it (fails) → implement → run it (passes) → commit.
- No placeholders ("add validation here", "TBD"). If you can't write the step concretely, the design isn't done.

Save plans where the repo keeps docs (e.g. `docs/plans/YYYY-MM-DD-<feature>.md`) unless the user prefers otherwise.

## 4. Execute

- Work task by task, marking progress. Follow the plan; if reality contradicts it, stop and raise it instead of silently improvising.
- Commit after each green task — small commits make review and rollback cheap.
- For independent tasks or independent failures, parallel agents can help — see `review-and-agents.md`.
- Get a review before integrating: after each task for risky work, or of the whole branch for cheap work.

## 5. Finish the branch

1. Run the full suite (and the tier of e2e/visual checks the change deserves). Red → report failures and stop; the options come after green.
2. Confirm the base branch — merging into the wrong base is expensive to undo.
3. Offer the choices and wait: merge locally / push and open a PR / keep the branch as is. Discarding work only happens on an explicit request, confirmed.
4. Push and merge are **separate, per-instance authorizations** in this playbook (see `SKILL.md`). A "yes" to push is not a "yes" to merge, and last time's yes is not this time's.
5. After a confirmed merge: fast-forward the local base, re-run tests on the merged result, then clean up worktree and branch (local and remote) — unless the user wants to keep iterating there.
