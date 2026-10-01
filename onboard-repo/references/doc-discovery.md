# Doc Discovery Procedure

## If a local clone is available

Read directly from disk — it's faster and avoids API rate limits. Check these directories (in order):

```
docs/
.github/
.github/instructions/
```

Collect every `.md` file found under them.

## If reading from GitHub (no local clone)

Use the `gh` CLI (preferred) or the GitHub API to list and fetch files:

```bash
# List a directory
gh api repos/{owner}/{repo}/contents/docs --jq '.[].name'

# Fetch a single file
gh api repos/{owner}/{repo}/contents/docs/FILENAME.md --jq '.content' | base64 -d
```

If a GitHub MCP server or similar tool is available in your environment, prefer its `get_file_contents`-style tool over raw `gh api` calls — same discovery order applies.

## Fallback discovery

If `docs/` does not exist, check alternative locations:
- `doc/`
- `documentation/`
- Root-level `*.md` files (beyond just `README.md`)

If no documentation is found at all, **stop and inform the user** rather than guessing from the source code alone.

## Always fetch these files

Regardless of directory contents, always fetch:
- `README.md` — high-level context
- `.github/copilot-instructions.md` — architectural overview (if present)
- `CLAUDE.md` or `AGENTS.md` — additional AI-assistant context (if present)

## Categorization

As you read each file, extract and label content for the two outputs:

**For knowledge-index (routing table):**
- Feature areas covered by each doc
- Source doc file paths (for linking)
- Key things worth capturing when writing stories/specs for each area

**For domain-glossary:**
- Domain concepts and definitions
- System entities (class names, collection names, service names)
- User roles and auth mechanisms
- External integrations
- Workflow terms and lifecycle states
- Important field names and their meaning
