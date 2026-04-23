# repograph

> Codebase knowledge vaults for coding agents. Conversation-driven generator, auto-activating consumer skill, drift-aware by design.

Coding agents work better when they have a map of the codebase they're working on — the invariants, the architectural rules, the "why" behind the code that lives in people's heads rather than in the source. `repograph` is a Claude Code plugin that helps a team author that map quickly and keep it current.

Inspired by [ArsContexta](https://github.com/agenticnotetaking/arscontexta) — a conversation-driven second-brain generator for personal knowledge. `repograph` ports the same method (atomic concept docs, Maps of Content, wiki-links, subagent isolation on large vaults) from personal-KM to codebase context.

## What you get

- A **generator skill** (`generate-repograph`) that walks a contributor through a conversation and produces a `.repograph/` vault in their repo.
- A **consumer skill** (`using-repograph`) that helps Claude Code navigate the vault when working in a repo that has one.
- A **verify command** (`/repograph-verify`) that detects drift between vault claims and current code.
- A lean vault format: plain markdown, wiki-linked, agent-agnostic.

## Install

In Claude Code:

```
/plugin marketplace add rwaltenberg/repograph
/plugin install repograph@repograph
```

The first command registers this repo as a marketplace; the second installs the plugin from it. (`repograph@repograph` is `<plugin-name>@<marketplace-name>` — both happen to be "repograph" since this repo hosts a single plugin and uses itself as the marketplace.)

## Use it

**Generate a vault in your repo:**

```bash
cd ~/dev/your-repo
claude
```

Then in Claude Code: `/generate-repograph`. Claude walks you through an interview; the vault lands under `.repograph/`.

**Re-run with a different contributor (enhance mode):**

`/generate-repograph` again in the same repo. Claude detects the existing vault, offers `enhance` mode by default, and lets the new contributor add depth where they know more than the first pass.

**Check for drift:**

```
/repograph-verify
```

Walks the vault, checks each citation against the current code, writes `.repograph/drift-report.md`.

## Vault layout

```
.repograph/
├── hub.md              # top-level Map of Content — agents read this first
├── config.yml          # vault path + pointer-file list + plugin version
├── domains/            # domain-level MOCs
├── concepts/           # atomic concept docs with wiki-links
├── invariants/         # [must] / [never] / [prefer] / [avoid] rules
└── tasks/              # skill-command stubs for common work
```

Full format spec: [`docs/vault-spec.md`](docs/vault-spec.md).
Method rationale: [`docs/method.md`](docs/method.md).

## Status

V0.1 — first release, April 2026. Known non-goals and intentional cuts are documented in [`docs/vault-spec.md`](docs/vault-spec.md#non-goals). Contributions welcome; see the method rationale for design intent before proposing structural changes.
