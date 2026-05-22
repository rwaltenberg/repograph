# repograph

> Codebase knowledge vaults for coding agents. Conversation-driven generator, auto-activating consumer skill, drift-aware by design.

Coding agents work better when they have a map of the codebase they're working on — the invariants, the architectural rules, the "why" behind the code that lives in people's heads rather than in the source. `repograph` is a set of agent skills plus a drift-detection command that helps a team author that map quickly and keep it current.

Works with any agent that reads the [open agent skills](https://github.com/vercel-labs/skills) format — Claude Code, Cursor, Codex, Aider, and others.

Inspired by [ArsContexta](https://github.com/agenticnotetaking/arscontexta) — a conversation-driven second-brain generator. `repograph` ports the same method (atomic concept docs, Maps of Content, wiki-links, subagent isolation on large vaults) from personal-KM to codebase context.

## What you get

- A **generator skill** (`generate-repograph`) that walks a contributor through a conversation and produces a `.repograph/` vault in their repo.
- A **consumer skill** (`using-repograph`) that tells the agent how to navigate the vault when working in a repo that has one — load the hub first, follow wiki-links on demand, honor invariants, surface verification markers rather than guess.
- A **verify workflow** (`repograph-verify`) that detects drift between vault claims and current code.
- A lean vault format: plain markdown, wiki-linked, agent-agnostic.

## Install

### Default — any agent, via `npx skills`

```bash
# Install both skills
npx skills add rwaltenberg/repograph

# Or install just one
npx skills add rwaltenberg/repograph --skill using-repograph       # consumer only
npx skills add rwaltenberg/repograph --skill generate-repograph    # generator only
```

This fetches SKILL.md files and drops them into your agent's skills directory (e.g., `.claude/skills/`, `.agents/skills/`, or whichever path your agent uses).

### Alternative — Claude Code plugin marketplace

If you're on Claude Code, you can install the full bundle (skills + slash commands) through the plugin system:

```text
/plugin marketplace add rwaltenberg/repograph
/plugin install repograph@repograph
```

This gets you the skills plus the `/repograph-verify` slash command registered natively.

## Use it

**Generate a vault in your repo:**

In an agent session inside your repo, ask the agent to run the generator:

> "Run generate-repograph"

The skill walks you through an interview; the vault lands under `.repograph/`.

**Re-run with a different contributor (enhance mode):**

Run the generator again in the same repo. It detects the existing vault, offers `enhance` mode by default, and lets a different contributor add depth where they know more than the first pass.

**Check for drift:**

- **Claude Code:** `/repograph-verify` (available as a slash command if you installed via the marketplace route).
- **Other agents:** ask the agent to follow the instructions in `commands/repograph-verify.md`.

Walks the vault, checks each citation against the current code, writes `.repograph/drift-report.md`.

## Vault layout

```text
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
