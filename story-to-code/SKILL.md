---
name: story-to-code
description: 'Generic pipeline that turns a story or feature request pasted directly in the prompt into working, tested code, in whatever repo/directory is currently open. Requires no ADO or live ticket-system fetch, so it is safe for interviews, take-homes, live demos, or quick prototypes in any codebase. If the target repo has already been onboarded (a ReferenceRepos/{repo}/knowledge-index.md + domain-glossary.md exist from the onboard-repo skill), reads them for grounding; otherwise proceeds with just the pasted story. Frames the story as a problem statement and hands it to the superpowers chain: brainstorming (design) -> writing-plans (spec + TDD plan) -> subagent-driven-development or executing-plans (implementation) -> finishing-a-development-branch (merge/PR). Use when the user pastes a story/ticket/feature description and asks to "turn this into code", "brainstorm and implement this", "spec, plan, and build this", or invokes /story-to-code.'
argument-hint: 'Paste the story/ticket text directly (title + description + ACs), or just describe the feature in plain language'
---

# Story → Code (generic)

Turn a pasted story or feature request into working code in the current repo by chaining the superpowers brainstorming → plan → implementation skills — no ADO or eBillingHub-specific context required.

## When to Use This Skill

- User pastes a story/ticket/feature description (from ADO, Jira, a GitHub issue, or just typed text) and wants it turned into code
- Interview, take-home, or live-demo scenario: someone hands over a problem statement and expects a working, tested implementation
- Quick prototyping in any repo — onboarded into this workspace or not
- Explicitly invoked as `/story-to-code`

**Not for:** fetching an ADO work item automatically or eBillingHub multi-repo scope *detection* across several engineering repos at once — use `story-to-spec-and-plan` for that (it also stops at plan; this skill goes all the way to code, for one repo at a time). Not for PM-facing acceptance criteria — use `write-ac` / `write-business-ac`.

## Prerequisites

- The `superpowers` plugin (`brainstorming`, `writing-plans`, `executing-plans` / `subagent-driven-development`, `finishing-a-development-branch`, `requesting-code-review`, `test-driven-development`). If it isn't installed, see Gotchas for a manual fallback.
- A git repository at the working directory the story targets — the chain commits as it goes.
- **Optional but recommended:** run `onboard-repo` on the target repo beforehand to generate `ReferenceRepos/{repo}/knowledge-index.md` + `domain-glossary.md` in this workspace. Step 2 reads them if present — richer grounding produces a better design in Step 3. Not required; the skill still works from the pasted story alone.

## Workflow

### Step 1 — Capture and frame the story

Take whatever the user pasted (structured ticket or plain text) and reduce it to one paragraph before invoking brainstorming: what's being asked, who it's for, and any constraints/ACs already given. Don't invent scope that wasn't stated — surface it as an open question instead of guessing.

If the input is only an ID with no text (e.g. "AB#12345", "JIRA-42") and no ticket system is connected in this session, ask the user to paste the actual title/description/ACs. This skill has no ticket-fetching integration by design — that's `story-to-spec-and-plan`'s job within its ADO+eBillingHub scope.

### Step 2 — Confirm target repo/directory, then load onboarding context if it exists

Default to the current working directory. If the user has multiple repos open (common when demoing), ask once which one this story targets.

Derive the repo short name (the directory name) and check whether it has been onboarded into this workspace: look for `ReferenceRepos/{repo-short-name}/knowledge-index.md` and `ReferenceRepos/{repo-short-name}/domain-glossary.md`.

- **If both exist:** read them in full. `knowledge-index.md` maps feature areas to source-doc paths — use it to route the story to the right area of the codebase. `domain-glossary.md` gives entity/service/component names — use it to speak the repo's actual vocabulary instead of guessing terms. Carry this context into Step 3 alongside the framed story.
- **If they don't exist:** proceed without them — don't block on this. If the repo is a real, ongoing project (not a one-off take-home/interview problem) and the user wants richer grounding, mention that running `onboard-repo` first would generate them, but let the user decide; don't run it unprompted.

### Step 3 — Hand off to `superpowers:brainstorming`

Announce: "I'm using the brainstorming skill to design this before writing any code." Provide the framed story from Step 1 as the seed input.

Brainstorming is self-driving from here: it explores project context, asks clarifying questions one at a time, proposes 2-3 approaches, gets design approval section by section, writes the spec, and its own terminal state invokes `superpowers:writing-plans`. Let it run to completion — do not shortcut the clarifying-question loop even if the story looks simple (see Gotchas).

### Step 4 — Let the native chain carry through to code

`writing-plans` writes the TDD-based implementation plan, then asks the user to pick an execution mode:

- **Subagent-driven** (`superpowers:subagent-driven-development`) — recommended when subagents are available; fresh subagent per task with two-stage review (spec compliance, then code quality)
- **Inline** (`superpowers:executing-plans`) — batch execution with checkpoints, for a parallel/separate session

Both paths end in `superpowers:finishing-a-development-branch`, which verifies tests pass and presents merge/PR/keep/discard options. Don't invent a different execution path — these are the only two the chain supports.

### Step 5 — Optional: an extra review pass

If the user wants one more look before merging (a common ask mid-interview — "would you also review this?"), run `superpowers:requesting-code-review` against the full diff before accepting Step 4's finishing options.

## Gotchas

- **Do not skip brainstorming's clarifying questions "because the story is simple."** Brainstorming hard-gates on this for a reason — unexamined assumptions in a short problem statement are exactly where wasted work happens. Simple stories still get a design, even if it's two sentences.
- **Do not add ADO/Jira/GitHub Issues fetch logic to this skill.** If the user wants that, point them at `story-to-spec-and-plan` instead of bolting fetch logic onto this generic one — keeping the two skills' scopes separate is what makes this one portable across interviews/repos.
- **If the superpowers plugin isn't installed**, invocations fail with something like `Unknown skill: superpowers:brainstorming`. Fall back manually using [`references/interview-spec-template.md`](references/interview-spec-template.md) and [`references/interview-plan-template.md`](references/interview-plan-template.md): write a short design note yourself (goal, approach, risks, open questions) and get sign-off, then a bite-sized TDD task list, then implement task-by-task with a failing test before each change. This is strictly lower quality than the real chain (no two-stage review) — say so explicitly rather than silently degrading.
- **Never start implementation on `main`/`master` without explicit consent** — inherited hard rule from `executing-plans`/`subagent-driven-development`. Create a feature branch first unless the user says otherwise.
- **Don't report a step as "done" without running its verification.** `finishing-a-development-branch` gates on tests passing at the end, but that doesn't cover earlier claims — if you say the design or plan step is complete, that claim still needs evidence (user approval captured, plan self-review actually run), not just "should be fine."

## Troubleshooting

| Issue | Solution |
|---|---|
| Story text is too vague to design against | Ask 1-2 clarifying questions yourself before invoking brainstorming — don't let brainstorming discover the vagueness after it has already started writing a design |
| User wants spec+plan only, no code (e.g. handing off to someone else) | Stop after Step 4's plan is written; skip the execution-mode question and don't invoke `executing-plans`/`subagent-driven-development` |
| No subagents available in this environment | Choose `executing-plans` (inline) instead of `subagent-driven-development` at the Step 4 prompt |
| Target directory has no git history yet | Run `git init` and an initial commit before Step 3, so `finishing-a-development-branch` has a base to diff against |
| `ReferenceRepos/{repo}/knowledge-index.md` exists but looks stale (repo has moved on since it was generated) | Say so to the user and ask whether to re-run `onboard-repo` before continuing, or proceed with the pasted story only |

## References

- `onboard-repo` (this repo) — generates `ReferenceRepos/{repo}/knowledge-index.md` + `domain-glossary.md`; run it first for richer grounding, though it's optional
- `superpowers:brainstorming` — design step, hard-gates on user approval before any code
- `superpowers:writing-plans` — TDD plan format, bite-sized tasks, execution-mode handoff
- `superpowers:subagent-driven-development` vs `superpowers:executing-plans` — pick based on subagent availability
- `superpowers:finishing-a-development-branch` — merge/PR/keep/discard decision
- `story-to-spec-and-plan` (this repo) — ADO-integrated counterpart that detects scope across *multiple* engineering repos and stops at plan instead of implementing
- [`references/interview-spec-template.md`](references/interview-spec-template.md) / [`references/interview-plan-template.md`](references/interview-plan-template.md) (ships with this skill) — generic, company-agnostic spec/plan worksheets for the manual fallback above
