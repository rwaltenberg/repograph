# Vault specification

The vault is plain markdown under `.repograph/` at the target repo root. Agent-agnostic: any tool that reads markdown can use it.

## File layout

```
.repograph/
├── hub.md
├── config.yml
├── domains/
│   └── <domain-id>.md
├── concepts/
│   └── <concept-id>.md
├── invariants/
│   └── <invariant-id>.md
├── tasks/
│   └── <task-id>.md
└── drift-report.md          # written by /repograph-verify; may or may not exist
```

IDs are kebab-case, stable identifiers. File name equals `<id>.md`.

## config.yml

```yaml
vault_version: 1
plugin_version: 0.1.0
vault_path: .repograph
pointer_files:
  - AGENTS.md
  - CLAUDE.md
```

`pointer_files` records which root instruction files the generator patched. Used by re-run detection and future migrations.

## Hub MOC (`hub.md`)

Entry point. Agents read this before any non-trivial code work.

Required sections:
- **What this repo is** — one paragraph.
- **Domains** — bulleted list of `[[domains/<id>]]` links with one-line descriptions.
- **Start here for common tasks** — optional, links to `[[tasks/<id>]]`.
- **Invariants at a glance** — bulleted list of `[[invariants/<id>]]` links, sorted by severity (hard before soft).
- **How to navigate** — standard block telling agents the rules (load concepts on demand, use subagents for multi-concept tasks, honor `[never]` strictly, surface `TODO: verify` rather than guess).

## Domain MOC (`domains/<id>.md`)

Frontmatter:
```yaml
---
id: data
title: Data Layer
contributors:
  - name: <contributor>
    date: YYYY-MM-DD
    role: <role>
---
```

Body:
- **What's in this domain** — paragraph.
- **Concepts** — bulleted list of `[[concepts/<id>]]` links.
- **Invariants** — bulleted list of `[[invariants/<id>]]` links scoped to this domain.
- **Common tasks** — optional list of `[[tasks/<id>]]` links.

## Concept doc (`concepts/<id>.md`)

Frontmatter:
```yaml
---
id: data-store
title: dataStore
domain: data
status: verified            # verified | unverified | needs-review
source_files:
  - path: <relative/path/to/file>
    symbol: <symbol-name>        # preferred; the exported name in the file
    line_range: 42-68            # fallback when no useful symbol exists
    hash: sha256:<hex>           # SHA256 of the cited region, whitespace-normalized
related:
  - <other-concept-id>
last_checked: YYYY-MM-DD
contributors:
  - name: <contributor>
    date: YYYY-MM-DD
    role: <role>
---
```

Body sections:
1. `# <title>`
2. `## What it is` — one-sentence summary, then 1–2 paragraphs.
3. `## Shape` — types/interfaces from source, with `path#symbol` or `path:line-line` citations.
4. `## How it's used` — real usage examples.
5. `## Invariants` — wiki-links to applicable invariants.
6. `## Related` — upstream/downstream concept links.
7. `## Historical context` — optional; from PR mining. Omit if empty.

Lines per doc: under 200.

## Invariant doc (`invariants/<id>.md`)

Frontmatter:
```yaml
---
id: never-import-mapbox-at-module-level
kind: never                 # must | never | prefer | avoid
severity: hard              # hard | soft
source_files:
  - path: <relative/path/to/file>
    symbol: <symbol>
    hash: sha256:<hex>
contributors:
  - name: <contributor>
    date: YYYY-MM-DD
    role: <role>
---
```

Body sections:
1. `# [<kind>] <imperative statement>` — e.g., `# [never] Import Mapbox GL at module level`.
2. Short prose explaining the rule.
3. `## Correct` — code example pulled from real file.
4. `## Incorrect` — synthetic counter-example.
5. `## Why` — reason the rule exists.

## Task stub (`tasks/<id>.md`)

Frontmatter:
```yaml
---
id: add-a-filter
title: Add a new filter
applies_to:
  - <domain-id>
---
```

Body: numbered steps naming files to touch and invariants to honor, with `[[wiki-links]]` to concepts and invariants.

## Wiki-link syntax

- `[[concepts/foo]]` — link to `concepts/foo.md`
- `[[invariants/bar]]` — link to `invariants/bar.md`
- `[[domains/baz]]` — link to `domains/baz.md`
- `[[tasks/qux]]` — link to `tasks/qux.md`

No leading `/`. Rendered correctly in Obsidian and GitHub; degrades gracefully in plain markdown viewers.

## Verification markers

Four tags, embedded anywhere in body content:

- `[unverified]` — claim could not be confirmed against code at write-time.
- `[needs-review]` — contributor flagged this for a more knowledgeable person.
- `TODO: verify with <role>` — explicit hand-off marker.
- `[stale]` — set by `/repograph-verify` when drift is detected.

Consumer skill is instructed to surface these rather than silently trust them.

## Content-hash convention

`hash: sha256:<hex>` stores the SHA256 of the cited region:
- For a symbol citation: the text from the symbol's start line through its end line.
- For a line-range citation: the text of those lines inclusive.
- Before hashing: strip trailing whitespace from each line; collapse runs of blank lines to one (via `cat -s`). This suppresses cosmetic-change noise.

The `/repograph-verify` command rehashes and compares.

## Non-goals (V0.1)

- No AST walkers or full-repo auto-ingestion.
- No auto-regen on drift — reporter only.
- No concurrent multi-contributor runs (first wins until they finish).
- No cross-repo vault linking.
- Some SKILL.md body wording (e.g., references to "the Agent tool" for subagent dispatch) is written in Claude Code terminology. Other agents can follow the same instructions but the phrasing may need light adaptation. The vault format itself is fully agent-agnostic.
