---
name: generate-repograph
description: Use when a user wants to generate or enhance a `.repograph/` knowledge vault for their repository. Orchestrates a conversation-driven interview that produces atomic concept docs, domain MOCs, invariants, and task stubs under `.repograph/`, grounded in existing docs and code.
---

# generate-repograph

You are running the `repograph` vault generator in the user's current repository. Your job is to produce (or enhance) a `.repograph/` vault through an interview with a contributor, grounded by existing repo docs, git history, and by reading source files to verify claims.

## Preflight

- Confirm the current working directory is a git repository: `git rev-parse --show-toplevel`. If not, stop and tell the user.
- Read the full interview script at `interview-script.md` (sibling file in this skill). Follow its phases.
- Read the templates in `templates/` (`hub.md.template`, `domain.md.template`, `concept.md.template`, `invariant.md.template`, `task.md.template`). Use them as the emission format.
- Read the vault spec at `../../docs/vault-spec.md` (relative to this skill) so you know the exact frontmatter and file layout you must produce.

## Detect mode

Check `.repograph/hub.md`:

- **Exists → enhance mode.** Follow interview-script.md Phase 0 enhance branch; then 0.5 → 6 → 8 → 9 → 10. Do NOT do the linear first-run walk.
- **Absent → first-run mode.** Follow interview-script.md Phase 0.5 → 1 → 2 → 3 → 4 → 5 → 7 → 8 → 9 → 10 in order.

## Execute the interview

Follow the phases in `interview-script.md`. At each phase:

- Ask questions one at a time. Do not batch.
- Adapt phrasing to the interviewee's vocabulary. If they say "store," use "store"; if they say "reducer," use "reducer."
- When a phase calls for reading source files, read them. Do not guess content — cite by path + symbol or path + line range.
- When the interviewee describes something you cannot find in code, ask for clarification or mark `[unverified]`.
- When the interviewee says "I don't know," write `TODO: verify with <role>` rather than invent.
- Skip phases the interviewee explicitly declines. Record skipped phases in a note at the end.

## Emit files with the right frontmatter

For every concept, invariant, domain MOC, and task stub you emit:

1. Compute the content hash for each `source_files` entry. The hash is SHA256 of the cited region (symbol body or line range), with trailing whitespace stripped per line and runs of blank lines collapsed to one.

   Use a bash command. Example for a line range:
   ```bash
   sed -n '42,68p' route-builder-web/src/stores/dataStore.ts \
     | sed 's/[[:space:]]*$//' \
     | cat -s \
     | sha256sum | cut -d' ' -f1
   ```

   For a symbol (e.g., an exported class `DataStore`), use `grep -n` to find the start line, then find the closing brace (for TypeScript/JS: scan forward for balanced braces; if the symbol is not a brace-delimited block, use the symbol's start line to the next top-level export). This is best-effort — if resolving the exact end is hard, fall back to a line range and record that instead.

2. Set `last_checked: <today>` (YYYY-MM-DD).

3. Append the contributor to `contributors:` with name, date, role.

4. Use wiki-link syntax — `[[concepts/foo]]`, `[[invariants/bar]]`, etc. — in body content.

## Non-hallucination discipline

**Before you commit anything:**

- Every code example in the vault must be a direct quote from a real source file you read. If you cannot cite it, do not emit it.
- Every file path, symbol name, PR number, and commit SHA in the vault must come from reading the actual repo or running a real command. No fabrication.
- Every `[unverified]` / `[needs-review]` / `TODO: verify` marker must correspond to a real gap in what you know — do not sprinkle them cosmetically.

If you find yourself tempted to generate content you cannot verify, stop and ask the interviewee instead.

## Pointer-file patching (Phase 9)

Detect these files at repo root, in order:

1. `AGENTS.md`
2. `CLAUDE.md`
3. `GEMINI.md`
4. `.cursorrules`
5. `.github/copilot-instructions.md`

For each that exists, patch it:
- Check if the existing file contains the string "repograph" — if yes, skip (idempotent).
- If no, append the pointer stanza from the interview script Phase 9.

If NONE exist, create `AGENTS.md` with the stanza as its whole content.

Record patched files in `.repograph/config.yml`:

```yaml
vault_version: 1
plugin_version: 0.1.0
vault_path: .repograph
pointer_files:
  - AGENTS.md       # only list files you actually patched
git_history_mined: true | false
```

## Commit discipline

Stage everything with `git add`, not `git add -A` (avoid accidentally adding unrelated changes).

Stage:
- `.repograph/` (all files)
- Any pointer files you patched or created (`AGENTS.md`, etc.)

Show the user a summary before committing. Ask for commit-message override; default to:

```
feat: add repograph knowledge vault (v0, contributor: <name>)
```

Do not commit until the user confirms.

## If the interview is interrupted

If the user says "pause" or the conversation stops:
- Do NOT leave partial files uncommitted in the vault; either finish the current phase or revert in-progress writes.
- Tell the user they can re-run `/generate-repograph` later; it will detect the vault and offer enhance mode.

Do not try to persist interview state to disk — `resume` mode is explicitly out of V0.1 scope.
