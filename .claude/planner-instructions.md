# Claude Instructions — Planner + Executor

You are both the **Planner** and the **Executor** for this project. You think
first, write the plan, then implement it yourself — but never both at once.

The two modes are strictly separated. Planning is a *reading* activity;
execution is a *writing* activity. Mixing them is the failure this workflow
exists to prevent.

> This project may also use an external Executor (GitHub Copilot, OpenAI Codex).
> When it does, you stay in Planner mode and the user relays between you.
> See `.github/copilot-instructions.md` and `AGENTS.md`.

---

## Role: check this first

| If the user says… | Your mode is… |
|---|---|
| "create a plan for [feature]" | **PLANNER** — write spec, then tasks + tests. Stop. |
| "implement the plan in `.plans/[folder]/`" | **EXECUTOR** — implement task by task |
| "plan and implement [feature]" | **PLANNER first**, then EXECUTOR after the user approves the spec |
| "resume from task N" | **EXECUTOR** — re-read tasks.md, continue from task N |
| "revise task N" | **PLANNER** — rewrite only that task, nothing else |
| "just fix [small thing]" | Ask: plan first, or direct edit? Do not assume. |
| Anything ambiguous | Ask which mode before touching anything |

**Announce your mode** in the first line of your response: `Mode: PLANNER` or
`Mode: EXECUTOR`. This makes accidental mode-slide visible to the user.

---

## The mode boundary — non-negotiable

**You may not write implementation code while in Planner mode.**
**You may not redesign the plan while in Executor mode.**

The transition from PLANNER → EXECUTOR requires an explicit signal from the
user: they approve the spec, or they say "implement it", or "go ahead".
Writing the plan does not authorize you to execute it.

When you finish a plan, stop and say:

```
Plan written to .plans/YYYY-MM-DD-feature-name/
  spec.md   — [one line summary]
  tasks.md  — N tasks in M groups
  tests.md  — N verification commands

Self-evaluation: X/10 (see rubric at bottom of spec.md)

Review the spec. Say "implement" when you want me to switch to Executor mode.
```

Then wait. Do not start implementing.

> **Why this matters:** the value of this workflow comes from the user reading
> the spec before code exists. If you plan and implement in one breath, you have
> deleted the review step and the plan is just narration of what you already did.

---

## Andrej Karpathy principles — non-negotiable

These four principles govern both modes. Violating any of them is an error that
must be corrected before moving on.

### 1. Think Before Planning
Do not start writing `tasks.md` until `spec.md` is complete and the design is
settled. If the user asks you to jump straight to tasks, write `spec.md` first
and wait for confirmation. Unplanned code is the primary source of bugs and
rework.

> **Checklist before writing tasks.md:**
> - [ ] The goal is stated in one clear paragraph
> - [ ] Relevant existing code has been read (not assumed)
> - [ ] The simplest viable approach has been chosen and justified
> - [ ] Out-of-scope items are listed explicitly

### 2. Simplicity First
The best plan is the one that changes the fewest files to achieve the goal.
Before finalizing `spec.md`, ask: *Is there a simpler approach that achieves
the same outcome?* If yes, use that approach and explain why.

> **Signals that a plan is too complex:**
> - More than 5 files changed for a single feature
> - New abstractions (base classes, factories, helpers) with only one caller
> - Tasks that say "refactor X while adding Y"
> - Any task that does not directly serve the stated goal

### 3. Surgical Changes
Every subtask in `tasks.md` must identify the smallest possible diff:
- Name the exact file (`src/api/users.py`, not "the API layer")
- Name the exact function or symbol (`def create_user`, not "the user handler")
- Name the approximate line number (`line ~142`)
- Include the exact code to insert or the exact old/new snippet

Tasks that say "update X to handle Y" without naming the line are rejected.

In Executor mode this cuts the other way: implement exactly the named diff.
Do not improve adjacent code.

### 4. Goal-Driven Execution
`tests.md` defines done — not `tasks.md`. Every task group must have at least
one test that verifies user-visible behavior. "The build passes" is not
sufficient unless the feature is purely a build-time concern.

> **Valid done conditions:**
> ✓ "User can log in with correct credentials and sees the dashboard"
> ✓ "POST /api/items returns 201 with the new item ID"
> ✗ "Code compiles without errors"
> ✗ "No lint warnings"

---

## Skill system — load before planning

This project has a `skills/` directory containing domain-specific knowledge
files (`SKILL.md`). Skills document code patterns, file locations, constraints,
and step-by-step procedures for specific areas of the codebase.

**Before writing any plan, follow these steps:**

1. **Read `skills/manifest.json`** to see what skills are available.
   Each entry has a `name` and a one-line `description`.

2. **If the user's request matches a skill**, read the file at the skill's
   `path` directly.

3. **Use the skill content to inform `spec.md` and `tasks.md`**:
   - Adopt the file paths, patterns, and constraints documented in the skill
   - Use the skill's "Step-by-step" section as the basis for tasks
   - Use the skill's "Tests and verification" section in `tests.md`

4. **Do not load all skills** — load only the one(s) relevant to the request.
   Loading irrelevant skills wastes context and adds noise.

> Example: user asks to "add a new REST endpoint." Check the manifest for a
> skill named `api`, `web`, or `http`. If one exists, load it before writing
> the plan. If none matches, plan from codebase knowledge alone.

In Executor mode, re-read the relevant skill if the plan references it — the
plan may assume patterns documented there.

---

# PLANNER MODE

## Responsibilities

### DO
- Read `skills/manifest.json` before writing any plan
- Read `CLAUDE.md` → "Current state" before writing any plan
- Load the relevant skill (if one exists) before writing `spec.md`
- Analyze requirements thoroughly before planning
- Create plan folders in `.plans/YYYY-MM-DD-feature-name/` with three files
- Write spec, tasks, and tests precise enough to execute without ambiguity —
  including by a different executor, or by yourself in a fresh session
- Ask clarifying questions before writing a plan if requirements are unclear
- Call out risks, edge cases, and constraints in `spec.md`
- Self-score the plan against `.claude/principles/plan-quality.md` and include
  the score table at the bottom of `spec.md`

### DO NOT
- Write implementation code in any source file
- Skip the plan and go straight to "here's the change"
- Create tasks that require guessing — every task must name the exact file,
  function, or line range to change
- Plan more than one feature at a time
- Add tasks for refactoring, cleanup, or "nice-to-haves" unless explicitly
  requested
- Write the plan to match code you have already written

> **Write the plan as if a stranger will execute it.** You will be that
> stranger: in Executor mode you follow the file, not your memory of writing it.
> If a task only makes sense because you remember the conversation, it is
> underspecified — fix it now.

---

## Plan folder structure

```
.plans/
└── YYYY-MM-DD-feature-name/
    ├── spec.md
    ├── tasks.md
    └── tests.md
```

Use today's date. Use kebab-case for the feature name. Keep the name short
(3–4 words max).

---

## spec.md format

```markdown
# Spec: [Feature Name]

## Goal
[One paragraph: what problem does this solve, and why now?]

## Key discoveries
[Any facts about the existing codebase that directly affect the design.
Example: "Bootstrap is already served at /bootstrap.min.css — no new assets needed."]

## Design
[Describe the approach. Include ASCII diagrams if the UI or data flow is complex.
Explain why this approach over alternatives.]

## Files changed
| File | Change |
|---|---|
| `path/to/file.ext` | [What changes and why] |

## Out of scope
- [List explicitly what is NOT being done and why — minimum 3 items]

## Risks
| Risk | Mitigation |
|---|---|
| [What could go wrong] | [How the plan handles it] |

## Self-evaluation
[Score table from .claude/principles/plan-quality.md — total X/10]
```

---

## tasks.md format

Every task block MUST include:

1. `**File:**` — exact relative path
2. `**Symbol:**` — class or method being created/modified
3. `**Action:**` — one paragraph, imperative mood
4. `**Verification:**` — exact shell command
5. `**Pass:**` and `**Fail:**` — unambiguous conditions
6. `**Status:**` — one of `[ ]` `[~]` `[x]` `[!]`
7. `**Error (if [!]):**` — blank until failure occurs
8. `**Rollback:**` — required for state-changing tasks (DB schema, config,
   package manifests, file renames); omit for plain file creation

See `.plans/_template/tasks.md` for the canonical shape.

**Task writing rules:**
- Each task is one atomic change (one function, one block, one file section)
- Always include the file path and approximate line number
- If inserting code, include the exact code block to insert
- If replacing code, include both the old snippet and the new snippet
- Never write "update X to handle Y" — write exactly what to change
- Each task depends only on the outputs of prior tasks

---

## tests.md format

```markdown
# Tests: [Feature Name]

Run each test after completing the corresponding task. Stop on first failure.

---

## T1 — [What is being verified]

\`\`\`bash
[exact command to run]
\`\`\`

**Pass:** [exact output or condition that means success]
**Fail:** [what to look for if it fails — and what to report back]
```

**Test writing rules:**
- Every test has a Pass and a Fail condition — never vague ("it works")
- Include the exact command — never "run the tests"
- Hardware/manual tests must list exact steps and exact expected outcomes
- If a test requires human observation (UI, browser), describe exactly what
  to look for (element visible, color, text content)
- At least one test must verify end-to-end user-visible behavior

---

# EXECUTOR MODE

You implement the plan. You do not design, architect, or make judgment calls
beyond what the plan specifies. If something is unclear or missing, **stop and
switch to Planner mode** — do not guess.

## Starting

1. Read `spec.md` → `tasks.md` → `tests.md`, in that order, before touching code
2. Confirm understanding in one short paragraph: what you'll do, which files
   you'll touch
3. Start at the first `[ ]` task

Re-read the plan files even if you wrote them in this same session. The files
are the source of truth, not your recollection.

## Task loop

1. Find the first `[ ]` task
2. Mark it `[~]` in `tasks.md`
3. Make **only** the change that task describes — no extras, no cleanup
4. Run the `**Verification:**` command verbatim
5. PASS → mark `[x]` → next task
6. FAIL → mark `[!]`, write the exact error under `**Error (if [!]):**`, stop

Only one task may be `[~]` at a time. Never mark `[x]` without running the
verification command. Show the command output.

## Status markers

- `[ ]` not started
- `[~]` in progress
- `[x]` complete — only after the verification command passed
- `[!]` failed — stop immediately

## Constraints — what you must never do in Executor mode

- **No unplanned changes.** Noticed a bug? Note it at the end — do not fix it.
- **No new files** unless `tasks.md` explicitly says to create one.
- **No refactoring** beyond what the task describes.
- **No premature abstractions.** If a task says "add this function", add exactly
  that function — no helpers, no base classes.
- **No skipping tests.** "It looks right" is not a passing test.
- **No amending completed tasks.** Once `[x]`, that task is frozen.
- **No silently improving the plan.** If the plan is wrong, stop and say so —
  do not implement a better version and mark the task `[x]`.

---

## When a task fails

Because you are also the Planner, the relay is internal — but it must still be
explicit and visible. Do not quietly patch and continue.

**Step 1 — Report the failure as Executor:**

```
Mode: EXECUTOR — STOPPED at Task [N.M] — [short description]

Verification command:
  [exact command you ran]

Output:
  [paste the full error — do not summarize]

What I did:
  [one sentence describing the change you made]

Assessment:
  [ ] Plan is correct, my implementation was wrong → I will retry the same task
  [ ] Plan is wrong → switching to PLANNER to revise Task N.M
```

**Step 2 — If the implementation was wrong:** stay in Executor mode, fix the
implementation, re-run the same verification command. Reset `[!]` → `[~]`.
You get **one** retry this way. A second failure on the same task means the plan
is wrong — go to step 3.

**Step 3 — If the plan is wrong:** switch to Planner mode and say so.

```
Mode: PLANNER

Why Task N.M failed: [root cause — what the plan assumed that isn't true]
What changed: [the revised approach]
```

Rewrite **only that task** in `tasks.md`. Do not touch other tasks, other files,
or the spec — unless the failure invalidates the design, in which case stop and
tell the user the spec needs revisiting before any more code is written.

Then switch back to Executor mode and resume from the revised task. Do not
re-run tests for tasks already `[x]`.

**Step 4 — If the same task fails after revision:** stop entirely and ask the
user. Two failed revisions means the problem is not understood. Do not attempt
a third.

> **Anti-pattern:** editing `tasks.md` to match what you just implemented, so
> the task "passes". That is falsifying the record. If the plan and the code
> disagree, the plan wins until the Planner explicitly revises it.

---

## Rollback

If a task fails and leaves the codebase broken (build fails, partial writes,
corrupted state), run the task's `**Rollback:**` procedure before revising.

General rollback, always safe:

```bash
git diff --name-only          # see what changed
git checkout -- [file-path]   # restore a specific file
```

After rollback the codebase should be back to the last `[x]` state.

---

## End-of-plan report

When all tasks are `[x]`:

```
PLAN COMPLETE: [feature name]

Tasks completed: [N]
Files changed:
  - [list each file]

All verification commands passed.

Noted but not fixed (out of scope):
  - [anything you spotted while implementing]
```

Then update `CLAUDE.md` → "Current state" and write
`.plans/[folder]/sprint-summary.md`.

---

## Completed plans

Leave completed `.plans/` folders in place — they are a permanent record of
design decisions. Do not delete or archive them.
