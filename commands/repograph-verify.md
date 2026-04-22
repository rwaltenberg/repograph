---
description: Walk the .repograph/ vault in the current repo, check every source_files citation against current code, classify drift, and write .repograph/drift-report.md plus a stdout summary.
---

# /repograph-verify

Run drift detection on the current repo's `.repograph/` vault.

## Preflight

- Confirm cwd is a git repo: `git rev-parse --show-toplevel`. If not, stop.
- Confirm `.repograph/hub.md` exists. If not, stop with: "No vault found at `.repograph/hub.md`. Run `/generate-repograph` first."
- Read `.repograph/config.yml` to get `vault_path` if different from default.

## Discover vault docs

Collect all markdown files under `.repograph/concepts/`, `.repograph/invariants/`, `.repograph/domains/`, `.repograph/tasks/`. For each, parse the YAML frontmatter and extract the `source_files` list (if present).

Domains and tasks may not have `source_files` — skip those.

## For each source_files entry, classify

Inputs: `path`, optional `symbol`, optional `line_range`, `hash` (format `sha256:<hex>`).

Run one of four checks:

### 1. File missing

If `<repo-root>/<path>` does not exist:
- Check `git log --follow --name-only --format= -- <path> | head -20` to see if the file was renamed/moved.
- If the file appears in git history under a new path → classify as **`moved`** with `new_path: <path>`.
- If it does not → classify as **`missing`**.

### 2. Symbol unresolved (only if `symbol:` is present)

Attempt to locate the symbol in the file. Strategy:

- For TypeScript/JavaScript: `grep -nE "^(export )?(class|function|const|let|var|interface|type|enum) ${symbol}\\b" <path>` to find start line. Then find end line by scanning forward for balanced braces (for class/function/interface) or end-of-line (for const/let/var/type alias).
- For Python: `grep -nE "^(class|def) ${symbol}\\b" <path>` to find start line. End line is the last line at the symbol's indentation + 1.
- For other languages: if `grep -n "${symbol}" <path>` produces no match → symbol unresolved.

If the symbol cannot be located → classify as **`missing`** (with note: "symbol unresolved").

### 3. Hash matches

With the symbol or line range resolved, extract the cited region. Normalize it:

```bash
# For a line range 42-68 in file <path>:
sed -n '42,68p' <path> | sed 's/[[:space:]]*$//' | cat -s
```

Compute SHA256 of the normalized region:
```bash
<normalized_region> | sha256sum | cut -d' ' -f1
```

If it matches the stored `hash:` → classify as **`ok`**.

### 4. Hash differs

If the normalized hash differs → classify as **`drifted`**.

## Write drift-report.md

Create (overwriting) `.repograph/drift-report.md`:

```markdown
# Drift report

Generated: <ISO8601 timestamp>
Vault files checked: <N>
Source-file citations checked: <M>

## Summary

- ok: <count>
- moved: <count>
- drifted: <count>
- missing: <count>

## Details

### OK (<count>)

<list of `vault_doc → path#symbol`, collapsed if count > 20>

### Moved (<count>)

| Vault doc | Old path | New path |
|-----------|----------|----------|
| `concepts/foo.md` | `src/old/foo.ts` | `src/new/foo.ts` |

### Drifted (<count>)

| Vault doc | Citation | Last checked |
|-----------|----------|--------------|
| `concepts/bar.md` | `src/bar.ts#Bar` | 2026-04-22 |

### Missing (<count>)

| Vault doc | Citation |
|-----------|----------|
| `concepts/baz.md` | `src/baz.ts#Baz` |

## Recommended actions

- **Moved:** update `path:` in the vault doc to the new path. Hash may still be OK.
- **Drifted:** re-read the current source and update the vault doc's shape/usage sections. Bump `last_checked:`.
- **Missing:** decide if the concept is still relevant. If yes, re-ground it on current code. If no, delete the vault doc.

Re-run `/repograph-verify` after patching to confirm.
```

Also mark drifted/missing docs with a `[stale]` tag at the top of the body (after the frontmatter) in the respective vault files, so the consumer skill surfaces the status. Do NOT modify the doc otherwise — only append the `[stale]` tag if not already present.

## Print stdout summary

Emit to the user:

```
repograph-verify: checked N docs, M citations.
  ok: <count>
  moved: <count>
  drifted: <count>
  missing: <count>

Drift report: .repograph/drift-report.md
```

If any `drifted` or `missing` exist, add: "Re-ground affected docs by re-reading current source. See drift-report.md for details."

## Exit

If everything is `ok`, exit cleanly with the all-green summary and no [stale] markers touched.

If drift found, exit non-zero-equivalent (in a Claude Code command context, that just means the summary flags the drift and the report file is written). CI integration can parse the summary.
