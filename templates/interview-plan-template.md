# `<problem name>` — Implementation Plan

> **Purpose:** Generic TDD-style implementation plan template for a take-home, live-coding interview, or quick prototype. Use this as the fallback format when `superpowers:writing-plans` isn't available, or as a standalone worksheet.
>
> **Spec:** [interview-spec-template.md](interview-spec-template.md) *(link to the filled-in copy)*
>
> Break the design into small, independently-testable tasks. Each task should be committable on its own. Use checkbox (`- [ ]`) syntax to track progress.

---

## File Structure

Files to create:

| Path | Responsibility |
|---|---|
| *[path/to/file.ext]* | *[what lives here]* |

Files to modify:

| Path | Change |
|---|---|
| *[path/to/existing-file.ext]* | *[what changes]* |

---

## Task 1: <name — e.g. "Core data model">

**Files:**
- Create: *[path]*

- [ ] **Step 1: Write a failing test for `<behavior>`**

*[what the test asserts]*

- [ ] **Step 2: Implement the minimum code to make it pass**

- [ ] **Step 3: Verify**

Run: `<test command>`
Expected: *[all tests green, including previous tasks' tests]*

- [ ] **Step 4: Commit**

```bash
git add <files>
git commit -m "<type>: <summary>"
```

---

## Task 2: <name>

**Files:**
- Create/Modify: *[path]*

- [ ] **Step 1: Write a failing test for `<behavior>`**
- [ ] **Step 2: Implement**
- [ ] **Step 3: Verify** — Run: `<test command>`
- [ ] **Step 4: Commit**

---

*Add more tasks as needed, following the same pattern: failing test → implementation → verify → commit.*

---

## Self-Review (run after writing this plan)

**1. Spec coverage check:** every section of the design doc maps to at least one task. List the mapping:

- Design §*[n]* *[section name]* → Task *[n]* ✅

**2. Placeholder scan:** no "TBD" / "TODO" / "implement later" left in any task body.

**3. Consistency check:** file paths, function/class names, and test command are the same across every task that references them.

---

## Execution Handoff

Plan complete. Options:

1. **Task-by-task (recommended for interviews)** — implement one task at a time, running its tests before moving on, so there's always a working, tested state to fall back to if time runs out.
2. **Batch** — implement several tasks then run the full suite once, for speed when time is short.

Which approach?
