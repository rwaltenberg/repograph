# Interview script

Scripted-but-adaptive outline the generator follows. Not a rigid Q&A tree — Claude adapts phrasing to the interviewee's vocabulary and skips phases if the interviewee has already answered them.

## Phase 0 — re-run detection

**Check:** Does `.repograph/hub.md` exist in the target repo?

**If yes (enhance mode):**

Ask: "I see this repo already has a `repograph` vault. Current state: {N domains, M concepts, K invariants, T tasks; X docs marked [needs-review] or [unverified]}. Three options:

(a) **Enhance** (default) — you'll interview a new contributor and add depth. Safe, never deletes existing content.
(b) **Regenerate** — wipe the vault and start fresh. Destructive. Requires explicit confirmation.
(c) **Cancel**.

Which?"

- If (a) — continue to Phase 0.5 (contributor info), then skip to Phase 6 (enhance-mode menu).
- If (b) — require typed `regenerate` confirmation string, then `rm -rf .repograph/` and continue to Phase 1.
- If (c) — exit cleanly.

**If no:** proceed to Phase 1.

## Phase 0.5 — contributor info (both modes)

Ask: "Contributor name (optional, used for attribution)?" Record as `contributor.name`.

Ask: "Role — pod lead, senior dev, QA, other?" Record as `contributor.role`.

Record today's date.

## Phase 1 — bootstrap (silent)

Read the following files if they exist at the target repo root:
- `AGENTS.md`
- `CLAUDE.md`
- `README.md`
- `README`
- `package.json`
- `pyproject.toml`
- `Cargo.toml`
- `go.mod`
- `composer.json`

Form an initial hypothesis about the repo: languages, frameworks, top-level structure. Do not ask the user anything yet.

## Phase 2 — sanity check

Tell the user what you saw. "I read your CLAUDE.md / package.json / etc. and I understand this repo as: {summary}. Does that match? Anything I missed or got wrong?"

Accept corrections before moving on.

## Phase 3 — domains

Ask: "What are the top-level domains of this codebase? A domain is a coherent chunk of responsibility — e.g., 'data layer,' 'UI,' 'API surface,' 'infra.' Most repos have 3–6."

Record domain IDs and one-line descriptions.

Emit `hub.md` with the domain list and emit one skeleton `domains/<id>.md` per domain. Commit nothing yet — accumulate changes.

## Phase 4 — per-domain walk

For each domain, ask: "Walk me through the {domain} domain. What are the main pieces? What does an agent need to know to work there?"

As the interviewee describes concepts:
- Record each concept with an ID.
- Read the relevant source files (ask the interviewee to name files or find them via grep).
- Pull types, interfaces, function signatures directly from code into the concept doc's "Shape" section with `path#symbol` or `path:line-line` citations.
- Compute content hashes.
- If the interviewee's description contradicts what you read in code, ask which is correct; record the resolution.
- If the interviewee doesn't know a concept deeply, offer to generate it from code alone and mark `status: needs-review`.

Emit `concepts/<id>.md` per concept. Link concept docs from the domain MOC. Update hub.

**Skip opt-out:** if the interviewee says "skip this domain, I don't know it," generate skeleton concepts from code only, mark all `status: needs-review`, and move on.

## Phase 5 — invariants

Ask: "What should agents never do in this repo? What must they always do? Think of rules you've had to explain to someone new, or bugs you've fixed more than once. Rules like 'don't import X at module level,' 'always use Y for server state,' etc."

For each invariant:
- Record kind (`must` / `never` / `prefer` / `avoid`) and severity (`hard` / `soft`).
- Ask for a concrete example of correct usage — read it from source, cite path + symbol/line.
- Compute content hash.
- Ask for or generate a synthetic counter-example.
- Ask why the rule exists ("what breaks if someone violates it?") — record as the "Why" section.

Emit `invariants/<id>.md` per invariant. Link from applicable domain MOCs and hub.

## Phase 6 — enhance-mode menu (enhance mode only; Phase 4–5 become skippable targeted edits)

In enhance mode, replace the linear Phase 4–5 walk with a menu:

"What would you like to contribute? Options:
(a) Walk me through a specific domain.
(b) Focus on concepts marked [needs-review] or [unverified]. I have: {list}.
(c) Add invariants from your experience.
(d) Extend or correct a specific concept — tell me which.
(e) Add a task stub for common work.
(f) Done for now."

Per selection:
- Show the existing content in full.
- Ask for additions, corrections, or verifications.
- For verifications: "Can you confirm this claim? y/n/partial." If `y`, upgrade `status` from `needs-review`/`unverified` to `verified`; append contributor to `contributors:`. If `n`, ask for the correct version and update. If `partial`, leave marker, append a note.
- For conflicts: show existing vs new side by side, ask which is correct, resolve.
- Update `last_checked` on any touched doc.

Loop until user picks (f).

## Phase 7 — common tasks

Ask: "What will agents commonly be asked to do in this repo? E.g., 'add a new filter,' 'add a new API endpoint,' 'wire up a new map layer.' Aim for 3–6 common task patterns."

Per task:
- Record ID and title.
- Ask: "What files does this touch?" "What order?" "What invariants apply?"
- Emit `tasks/<id>.md` with steps, file list, invariant wiki-links.

## Phase 8 — git-history mining

If `gh` CLI is available AND the repo is on GitHub:

Run: `gh pr list --limit 30 --state merged --json number,title,body,mergedAt`

For each PR, skim the body for:
- Explanations of *why* a change was made
- Mentions of bugs that drove a design
- Invariants the PR establishes or violates

For any non-obvious historical context found, propose a "Historical context" note for the relevant concept or invariant. Show the note to the interviewee and ask: "Include this?" Append only if confirmed.

If `gh` is not available or the repo isn't on GitHub: skip the phase cleanly, note in config.yml.

## Phase 9 — pointer-file patching

Detect root instruction files in precedence order:
1. `AGENTS.md`
2. `CLAUDE.md`
3. `GEMINI.md`
4. `.cursorrules`
5. `.github/copilot-instructions.md`

For each that exists, check if the pointer stanza is already present (search for the string `repograph`). If absent, append:

```markdown

## Repository knowledge vault

This repo uses [repograph](https://github.com/rwaltenberg/repograph). Knowledge vault at `.repograph/`. Entry point: `.repograph/hub.md`.

### If you're authoring code in this repo

- Read `.repograph/hub.md` before non-trivial changes.
- Honor hard invariants under `.repograph/invariants/` — they are blocking.
- If you modify a file cited in any `.repograph/` doc, update that doc in the same change. If the update is out of scope for this PR, append a `[stale]` tag to the affected doc's body and surface to the reviewer.
- Run `/repograph-verify` after changes that touch load-bearing code.

### If you're reviewing code in this repo

- If the PR modifies any file cited in `.repograph/` and does not update the corresponding concept/invariant doc — flag it. The author should either update the doc, add a `[stale]` tag, or justify why the citation is still accurate.
- If the PR violates any hard invariant under `.repograph/invariants/` (kind: `never`/`must`, severity: `hard`) — block merge.
- When drift shows up in `.repograph/drift-report.md`, call it out.
```

If none of the instruction files exist, create `AGENTS.md` with the stanza as its initial content.

**`.github/copilot-instructions.md` exception:** instead of the full stanza, write a one-line pointer so all instructions stay canonical in `AGENTS.md`:

```markdown
Refer to [AGENTS.md](../AGENTS.md) for all repo instructions.
```

This avoids duplication drift between the author-facing file and the reviewer-facing file.

Record patched files in `.repograph/config.yml` under `pointer_files`.

## Phase 10 — staging + review

Stage all files with `git add`. Show the user a summary of what was created:

"Summary:
- N files under `.repograph/` (hub + N domain MOCs + N concepts + N invariants + N tasks + config)
- M files patched at repo root (pointer stanzas added)

Review the diff? (recommended)"

If yes, run `git diff --cached` and let the user scroll.

Ask: "Commit now? Suggested message: `feat: add repograph knowledge vault (v0, contributor: <name>)`."

On user confirmation, commit with that message (or the user's override).

## Non-hallucination rules

Enforce throughout:

- Every code example in a concept or invariant must come from a real file you read. Cite with `path#symbol` or `path:line-line`.
- If the interviewee asserts a claim you cannot verify from code, mark it `[unverified]` in the output.
- If the interviewee does not know a detail, write `TODO: verify with <role>` — never invent an answer.
- Never fabricate file paths, symbol names, or PR numbers.
- If you cannot find a symbol the interviewee named, ask for clarification rather than skipping or inventing.
