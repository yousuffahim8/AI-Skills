---
name: onboard-repo
description: 'Onboard an external repo as a documentation reference for AI-assisted development. Use when the user asks to onboard a repo, add a reference repo, or generate a knowledge-index/domain-glossary for a repo — given as owner/repo, a GitHub URL, or a local path. Discovers docs (local clone preferred, GitHub as fallback) and generates knowledge-index.md + domain-glossary.md.'
argument-hint: 'Repo to onboard: owner/repo, a GitHub URL, or a local path (e.g. octocat/hello-world)'
---

# Onboard Repository

Onboard an external repository into this workspace by reading its documentation and generating two reference files: a **knowledge-index** (routing table: feature area → source docs) and a **domain-glossary** (terminology, entities, roles, integrations). These let an AI coding assistant (Claude Code, GitHub Copilot, etc.) find the right source material for a repo it doesn't have open, instead of guessing from training data.

## Workflow

### Step 1 — Parse the repo input

Accept any of:
- `owner/repo` (e.g. `octocat/hello-world`)
- A full GitHub URL (e.g. `https://github.com/octocat/hello-world`)
- A local filesystem path to an existing clone

Derive `{repo-short-name}` from the last path segment. This becomes the workspace folder name.

### Step 2 — Discover documentation files

Prefer a local clone if the input was a path, or if a local clone is already known/configured for this repo — reading from disk is faster and avoids API rate limits. Otherwise fetch from GitHub.

Read [doc-discovery.md](./references/doc-discovery.md) for the full discovery procedure (which directories to scan, fallback locations, and what to categorize as you read).

### Step 3 — Read all documentation files

Read the full contents of every `.md` file discovered in Step 2. As you read, extract and categorize content for the two output files:
- **Knowledge-index:** feature areas, source doc paths, key details worth capturing per area
- **Domain-glossary:** entities, roles, integrations, workflows, field names

### Step 4 — Generate knowledge-index.md

Create `{repo-short-name}/knowledge-index.md` following [knowledge-index-template.md](./assets/knowledge-index-template.md).

### Step 5 — Generate domain-glossary.md

Create `{repo-short-name}/domain-glossary.md` following [domain-glossary-template.md](./assets/domain-glossary-template.md). Adapt sections to fit the repo's domain — omit empty sections, add new ones if needed.

### Step 6 — Create the files in the workspace

Create the folder and both files:
1. `reference-repos/{repo-short-name}/knowledge-index.md`
2. `reference-repos/{repo-short-name}/domain-glossary.md`

### Step 7 — Update your own project's config files (optional)

If this workspace maintains its own routing/instruction files (e.g. `CLAUDE.md`, `.github/copilot-instructions.md`, `README.md`) that list onboarded repos, update them following [config-update-rules.md](./references/config-update-rules.md). Skip this step if the workspace has no such files.

### Step 8 — Summary

Present a summary table to the user:

| Item | Status |
|---|---|
| Docs discovered | {count} files in `docs/`, {count} in `.github/` |
| Feature areas indexed | {count} rows in routing table |
| Glossary terms defined | {count} terms across all sections |
| Folder created | `reference-repos/{repo-short-name}/` |
| Config files updated | ✅ / skipped |

Ask the user to review the generated files and confirm if any adjustments are needed.
