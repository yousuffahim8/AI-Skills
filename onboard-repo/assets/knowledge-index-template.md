# Knowledge Index Template

Use this template when generating `{repo-short-name}/knowledge-index.md`. Replace all `{placeholders}` with actual values.

---

```markdown
# {repo-short-name}: Knowledge Index & Routing Table

This file tells an AI coding assistant **which source documents to read** for each feature area of this repo.

**Source of truth:** `https://github.com/{owner}/{repo}`
Never copy content from those docs into this file. Always link and fetch live.

---

## How to Use This Index

1. Identify the feature area your task touches from the table below
2. Read the linked source docs before writing any code or specs
3. The **Always load** row applies to every task. Read those files first

---

## Routing Table

| Feature Area | Source Docs to Read | Key things to capture |
|---|---|---|
| **Always load (every task)** | [file1](https://github.com/{owner}/{repo}/blob/main/path/file1.md) · [file2](...) | High-level context items |
| **{Feature area 1}** | [doc-link](...) | Key details for this area |

---

## Fetch Command

To read any of the source docs above without a local clone:

\`\`\`bash
# Fetch a docs/ file
gh api repos/{owner}/{repo}/contents/docs/FILENAME.md --jq '.content' | base64 -d

# Fetch a .github/ file
gh api repos/{owner}/{repo}/contents/.github/FILENAME.md --jq '.content' | base64 -d
\`\`\`

---

## Source Doc Inventory

### `docs/`: Documentation
| File | Topic |
|---|---|
| [filename](https://github.com/{owner}/{repo}/blob/main/docs/filename.md) | Brief topic description |

### `.github/`: Architecture & Integration
| File | Topic |
|---|---|
| [filename](https://github.com/{owner}/{repo}/blob/main/.github/filename.md) | Brief topic description |
```
