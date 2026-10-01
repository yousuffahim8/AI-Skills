---
title: <problem name> — design
date: <YYYY-MM-DD>
status: draft
authors: <your name>, <assistant name>
---

# `<problem name>` — design

> **Purpose:** Generic design-doc template for a take-home, live-coding interview, or quick prototype — no ticket system, no company-specific repo conventions. Use this as the fallback format when `superpowers:brainstorming` isn't available, or as a standalone worksheet you fill in by hand before writing code.
>
> Copy this file, fill in the placeholders, delete this callout and any section that's genuinely not applicable (say why if you drop one).

---

## 1. Problem Statement

*1–3 sentences: what is being asked, who it's for, and any constraints already given (time limit, language/framework requirement, "must handle X input").*

**Given:**
- *[fact/constraint 1]*
- *[fact/constraint 2]*

**Not given / assumed:** *[state assumptions explicitly rather than silently picking one]*

---

## 2. Scope

**In scope:**
- *[capability 1]*
- *[capability 2]*

**Out of scope (explicitly deferred):**
- *[thing you're deliberately not building, and why — e.g. "auth, since the prompt only asks for the core algorithm"]*

---

## 3. Approaches Considered

### Approach A — <name>

**Description:** *one paragraph*

**Pros:** *[...]*
**Cons:** *[...]*

### Approach B — <name>

**Description:** *one paragraph*

**Pros:** *[...]*
**Cons:** *[...]*

**Chosen approach:** *[A or B]* — *[1–2 sentence rationale — the deciding factor, not a restatement of the pros]*

---

## 4. Design

*Describe the shape of the solution concretely enough that someone else could implement it from this section alone.*

### Key components / interfaces

| Component | Responsibility | Notes |
|---|---|---|
| *[ClassName / module / function]* | *[what it does]* | *[key edge case or invariant]* |

### Data flow / control flow

1. *[step 1]*
2. *[step 2]*
3. *[step 3]*

### Edge cases

- *[empty input / zero / negative / duplicate / concurrent access / etc. — whatever applies to this problem]*

---

## 5. Testing Strategy

*What proves this works? List the test cases you intend to write, not just "unit tests".*

- *[test case 1 — input → expected output]*
- *[test case 2 — edge case]*

---

## 6. Open Questions

| # | Question | Status |
|---|---|---|
| 1 | *[anything you'd ask the interviewer/PM if you could]* | Open |

---

## Handoff

Design approved. Next: turn this into a task-by-task implementation plan (see [`interview-plan-template.md`](interview-plan-template.md)).
