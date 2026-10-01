# Config Update Rules

Optional step — only applies if your workspace maintains its own routing/instruction files that list onboarded reference repos. Skip entirely if it doesn't.

After generating the knowledge-index and domain-glossary files, update whichever of these exist.

## 1. `.github/copilot-instructions.md`

### Source of Truth section

Add the new repo as a bullet:

```markdown
- **{repo-short-name}** → `https://github.com/{owner}/{repo}` ({brief description})
```

### Files to Read section

Add two new rows to the table:

```markdown
| [`reference-repos/{repo-short-name}/knowledge-index.md`](../reference-repos/{repo-short-name}/knowledge-index.md) | Routing: {brief description} → source doc links |
| [`reference-repos/{repo-short-name}/domain-glossary.md`](../reference-repos/{repo-short-name}/domain-glossary.md) | {Brief description of glossary contents} |
```

---

## 2. `CLAUDE.md` / `AGENTS.md`

### Source-of-truth list

Add the new repo:

```markdown
- **{repo-short-name}** → `https://github.com/{owner}/{repo}`
```

### Files to read section

Add two new rows:

```markdown
| [reference-repos/{repo-short-name}/knowledge-index.md](reference-repos/{repo-short-name}/knowledge-index.md) | Routing: {brief description} → source doc links |
| [reference-repos/{repo-short-name}/domain-glossary.md](reference-repos/{repo-short-name}/domain-glossary.md) | {Brief description of glossary contents} |
```

### Glossary index (if the workspace keeps one)

Add the new repo's glossary to the list:

```markdown
- {Layer name}: [reference-repos/{repo-short-name}/domain-glossary.md](reference-repos/{repo-short-name}/domain-glossary.md)
```

---

## 3. `README.md`

### Repository Map section

Add the new repo folder to the tree under `reference-repos/`, following the existing pattern:

```
├── reference-repos/
│   ├── {repo-short-name}/
│   │   ├── knowledge-index.md         ← routing table: {brief description}
│   │   └── domain-glossary.md         ← {brief glossary description}
```
