---
name: using-repograph
description: Use when working in a repository that contains a `.repograph/` knowledge vault. Guides the agent to load the hub MOC first, follow wiki-links on demand, use subagent isolation for multi-concept tasks, honor invariants strictly, and surface verification markers rather than guess.
---

# Using a repograph vault

You are working in a repository that has a `repograph`-generated knowledge vault at `.repograph/`. This vault is a map of the codebase the team has authored for agents — it records architecture, invariants, gotchas, and common task patterns that aren't obvious from the code alone.

**Invoke this skill** when the current repo has `.repograph/hub.md` and you are about to do any non-trivial code work. "Non-trivial" = changes that touch more than a single isolated file, or any change that wires across layers (stores, APIs, UI).

## The rule

**Read `.repograph/hub.md` before touching code.** The hub names the repo's domains, lists invariants at a glance, and points to common-task stubs. It is short by design — usually under 100 lines.

From the hub, follow wiki-links to exactly the concepts and invariants your task needs. Do not preload the whole vault. Concept docs are designed to be loaded on demand.

## Six behaviors this skill enforces

### 1. Hub first

Before you propose changes or read a single source file related to a non-trivial task, read `.repograph/hub.md`. Use its "Domains," "Invariants at a glance," and "Start here for common tasks" sections to orient.

If the hub points you at a matching task stub under `tasks/<id>.md`, read it. Task stubs are vetted playbooks for common work and will save you from common mistakes.

### 2. Wiki-links on demand, not eagerly

Wiki-links look like `[[concepts/foo]]`, `[[invariants/bar]]`, `[[domains/baz]]`, `[[tasks/qux]]`. Resolve them to files under `.repograph/` only when you need the content.

Do not read every concept doc "just in case." Small context windows are a feature of this design — load what you need, leave the rest.

### 3. Subagent isolation for multi-concept tasks

If your task clearly spans 3 or more concepts (you can see this from the hub's task mapping or from following wiki-links out of a concept), spawn a fresh subagent per concept via the Agent tool. Each subagent:

- Gets a minimal prompt: "Read `.repograph/concepts/<id>.md`, read the cited source files, summarize the concept's shape and invariants in under 150 words."
- Returns only the summary, not raw quotes.

Collect summaries back in your main context. Then act. This matches the 6-Rs pattern from ArsContexta: isolation prevents attention degradation on larger vaults.

For tasks that span 1–2 concepts, read inline in your current context — don't spawn for small scopes.

### 4. Invariants are hard constraints

Files in `.repograph/invariants/` have frontmatter `kind` and `severity`. Treat them as follows:

- **`kind: never` + `severity: hard`** — absolute block. Do not violate. If your proposed change appears to require violating one, stop and raise it with the user explicitly; do not work around it silently.
- **`kind: must` + `severity: hard`** — required action. Your change is not complete until you satisfy it.
- **`kind: prefer` / `avoid` + `severity: soft`** — strong defaults. Follow unless you have a specific reason, which you surface to the user.

Before finalizing any change, list the invariants that apply to the files you touched and confirm you satisfied each.

### 5. Surface verification markers — never guess past them

Four markers exist in vault content. When you encounter one while reading a concept or invariant:

- **`[unverified]`** — the doc author couldn't confirm this claim at write-time. Spot-check it yourself by reading the cited source file.
- **`[needs-review]`** — a contributor flagged this for someone with more knowledge. Treat as suspect; verify from source or ask.
- **`TODO: verify with <role>`** — explicit hand-off. If you can verify, do so and offer to update the doc. If not, surface to the user.
- **`[stale]`** — `/repograph-verify` already flagged drift. Re-read the current source file; assume the doc is out of date.

Never silently rely on a marked claim as if it were verified.

### 6. Spot-check on stale docs

If a concept or invariant doc has `last_checked:` more than 90 days before today, briefly read the cited source files (or at minimum `git log -5` on them) before relying on the doc's claims. Note any drift you find.

## When code contradicts the vault

The vault can be wrong — code moves faster than docs. If you read source and see a direct contradiction with a vault claim:

1. Trust the current code.
2. Flag the contradiction explicitly in your reply to the user: "I noticed `.repograph/concepts/foo.md` claims X, but `src/foo.ts:42` actually does Y."
3. Offer to patch the vault (or run `/repograph-verify` to see if this is already detected).

Never pretend the vault is correct to avoid the friction. Staleness surfaced is the vault working as intended.

## Anti-patterns to avoid

- **Reading every file in `.repograph/` before starting.** The vault is designed for on-demand traversal. Eager loading burns context.
- **Treating concept docs as authoritative over current source.** Code is ground truth; the vault is a map. Spot-check citations for anything load-bearing.
- **Silently working around an invariant.** If you think you need to, stop and surface it.
- **Ignoring the hub because "I know this repo."** Pod leads who know the repo still benefit from the hub — it's a reminder of invariants that are easy to forget during focused work.

## Output cue

When you've consulted the vault for a task, mention it briefly in your response — something like "Consulted `.repograph/concepts/data-store.md` and `.repograph/invariants/never-import-mapbox-at-module-level.md`." This gives the user a quick audit trail and makes it obvious when the vault is or isn't in use.
