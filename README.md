# multitenant-saas-playbook

A [Claude Code](https://claude.com/claude-code) skill: an engineering playbook for building and evolving a real, paying-customer-facing multi-tenant SaaS. The reference stack is Next.js + Supabase/Postgres + Tailwind + Vercel, but the lifecycle and diagnostic habits generalize to other stacks.

It captures the operating discipline that came out of actually shipping a multi-tenant SaaS solo, over many rounds of: build → a real user tries it → something's subtly wrong → find the actual mechanism → fix it → check whether the same mechanism is lurking elsewhere. It is not a generic "how to code" guide — it's the specific residue: gotchas that cost real debugging time, design calls that turned out right, and calls that had to be reversed after real usage.

## What's inside

- [`SKILL.md`](SKILL.md) — the main playbook: push/merge authorization discipline, "verify by seeing it, not by inferring it," how to diagnose a "this is broken" report, auditing for the same bug shape after a fix.
- [`references/nextjs-supabase-gotchas.md`](references/nextjs-supabase-gotchas.md) — concrete bug patterns in the Next.js/Supabase/Postgres stack, each with a broken/fixed code pair.
- [`references/multitenant-architecture.md`](references/multitenant-architecture.md) — how to design one codebase that serves structurally different businesses without forking.
- [`references/responsive-design-method.md`](references/responsive-design-method.md) — a breakpoint strategy and mockup-first process for retrofitting responsiveness onto an existing app.

## Install

Clone it directly into your Claude Code skills folder:

```bash
git clone https://github.com/lean19r-cell/multitenant-saas-playbook ~/.claude/skills/multitenant-saas-playbook
```

On Windows, that's `%USERPROFILE%\.claude\skills\multitenant-saas-playbook` (or `C:\Users\<you>\.claude\skills\multitenant-saas-playbook`).

To use it inside a single project instead of globally, clone it into that project's `.claude/skills/multitenant-saas-playbook` and commit it — anyone who clones the project repo gets the skill automatically.

No build step, no registration — Claude Code auto-discovers any folder under `.claude/skills/` with a valid `SKILL.md`. Start a new session (or restart) after installing so it picks it up.

To update later: `git -C ~/.claude/skills/multitenant-saas-playbook pull`.
