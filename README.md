# Little Trinity – Spec Pack & Execution Guide

A spec-driven delivery pack for the Little Trinity home kids' library.
Copy the contents of this folder into the root of your repository (next to `api/` and `web/`).

```
CLAUDE.md                      ← rules the agent follows in every session (edit to taste)
docs/
  product/requirements.md      ← what we are building, business rules, assumptions
  architecture/overview.md     ← AWS architecture, module boundaries, conventions
  adr/                         ← architecture decision records
  specs/NNN-*/spec.md          ← one spec per feature slice, executed in order
```

## Execution order

| # | Spec | Outcome | Depends on |
|---|------|---------|-----------|
| 000 | Foundations & walking skeleton | Local stack runs (API + web + Postgres + Mailpit), CI green, conventions enforced | your scaffolding |
| 001 | Identity & family profiles | Parents register/login (Cognito), manage children, admin approves families | 000 |
| 002 | Catalogue & inventory | Admin adds books by ISBN, copies with barcodes; families search/browse | 001 |
| 003 | AWS infrastructure & CD to dev | Terraform + GitHub Actions deploy to a real **dev** environment | 002 (can be done any time after 000) |
| 004 | Collection/return slots | Daily 17:00 & 19:00 slots with capacity, closures | 001 |
| 005 | Reservations | Reserve anytime, copy held, collection slot booked | 002, 004 |
| 006 | Loans: check-out & check-in | Admin hands over / receives books at slot, due dates | 005 |
| 007 | Notifications | Transactional emails via outbox + SES | 005, 006 |
| 008 | Waitlist, renewals, overdue | Queue for unavailable books, renew, overdue handling | 006, 007 |
| 009 | Admin dashboard, reports & bulk import | Daily slot run-sheet, stats, CSV import, label printing | 006 |
| 010 | Production readiness & go-live | Security, privacy, a11y, observability, prod env, runbooks | all |

Each spec is sized to roughly 1–2 weeks of agent-assisted work, split into small tasks.

## The loop (repeat for each spec)

**Step 1 – Read & refine the spec (you).**
Read `docs/specs/NNN-*/spec.md`. Change anything you disagree with *before* any code exists. Resolve every item under "Open questions".

**Step 2 – Plan (Claude, plan mode – no file edits).**
Switch Claude Code to plan mode, then:
```
Read CLAUDE.md, docs/architecture/overview.md and docs/specs/NNN-xxx/spec.md.
Produce docs/specs/NNN-xxx/plan.md (design: packages, classes, DB migrations, endpoints,
React routes/components, test approach) and docs/specs/NNN-xxx/tasks.md (ordered checklist
of small tasks, each independently committable, each with its own verification step).
Flag anything in the spec that is ambiguous or conflicts with the existing code.
```
Review both files. Push back until you're happy. Commit: `docs(NNN): plan and tasks`.

**Step 3 – Implement one task at a time (Claude, normal mode – you approve edits).**
```
Implement task T1 from docs/specs/NNN-xxx/tasks.md only. Follow CLAUDE.md.
Write/extend tests first where practical. Run the verification commands and show results.
Stop when T1 is done and summarise what changed and why.
```
Review the diff in your IDE, ask "why" about anything unclear, then commit
(`feat(reservations): T1 add reservation entity and migration`). Tick the task in `tasks.md`.

**Step 4 – Verify the slice.**
```
All tasks for spec NNN are done. Verify every acceptance criterion in spec.md one by one:
point to the test(s) that prove it or run it manually against the local stack. List gaps.
```
Fix gaps, open a PR, let CI run, merge.

**Step 5 – Keep docs true.**
```
Update spec.md (status: Done), overview.md and ADRs if anything changed during implementation.
```

## Tips for staying in control
- Keep edit approval **on** (don't auto-accept) until you trust the patterns in a module; relax later.
- One task = one commit. If a task's diff is too big to review in ~10 minutes, ask Claude to split it.
- Start each new spec in a fresh Claude session – the repo docs carry the context, not the chat.
- Ask for explanations: "Walk me through the reservation flow from controller to DB."
- Use a second session as reviewer: "Review the diff on this branch against spec 005 and CLAUDE.md."
