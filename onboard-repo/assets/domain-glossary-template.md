# Domain Glossary Template

Use this template when generating `{repo-short-name}/domain-glossary.md`. Replace all `{placeholders}` with actual values. Adapt sections to fit the repo's domain. Omit sections with no relevant content, add new ones if the repo introduces concepts not covered here (e.g., "Report Types", "ETL Pipelines", "Notification Channels").

---

```markdown
# Domain Glossary: {Descriptive Title for This Repo}

Use these terms consistently when writing specs, stories, or code for this repo. Where a term maps to a real system entity, the entity name is shown in `code` format.

---

## Core Domain Concepts

| Term | Definition | System Entity |
|---|---|---|
| **{Term}** | {Definition} | `{EntityClass}` or n/a |

---

## Users & Roles

| Term | Definition | Auth Provider |
|---|---|---|
| **{Role}** | {Definition} | {e.g. OAuth / SAML / n/a} |

---

## System Integrations

| Term | Definition |
|---|---|
| **{Integration}** | {Definition} |

---

## Workflow Terms

| Term | Definition |
|---|---|
| **{Term}** | {Definition} |

---

## Important Field Names (for specs/stories involving data)

| Field | Entity | Meaning |
|---|---|---|
| `{fieldName}` | `{Entity}` | {Meaning} |
```
