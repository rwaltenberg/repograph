# Method: ArsContexta, adapted for codebases

`repograph` borrows its method from [ArsContexta](https://github.com/agenticnotetaking/arscontexta) — a Claude Code plugin that generates personalized knowledge systems from conversation. This doc records what we borrowed and what we changed.

## What ArsContexta does

ArsContexta generates a personal second-brain by interviewing the user about how they think and what they work on. Output is a three-space vault:

- `self/` — agent identity, methodology, goals
- `notes/` — the knowledge graph (wiki-linked markdown)
- `ops/` — operational state (queues, sessions)

Knowledge is organized as atomic notes (Zettelkasten), linked into Maps of Content (MOCs) at hub, domain, and topic levels. Processing commands (`/reduce`, `/reflect`, `/verify`, `/ralph`) spawn fresh subagents with isolated contexts to avoid attention degradation in large vaults. The method traces to 249 research claims from knowledge management, cognitive science, and agent design (Zettelkasten, Cornell Note-Taking, Evergreen, PARA, GTD, spreading activation, small-world topology).

## What we borrowed

1. **Conversation-driven generation.** The primary input is an interview, not a filesystem scan. A senior contributor describes the codebase; Claude verifies and fleshes out by reading code.
2. **Atomic concept docs + MOCs.** Each concept is one markdown file. Domain and hub MOCs route agents to concepts as needed.
3. **Wiki-links for graph traversal.** `[[concepts/foo]]` syntax lets agents walk the graph on demand rather than loading everything.
4. **Subagent isolation on large vaults.** When a task spans many concepts, the agent spawns one subagent per concept with an isolated context window, then collects summaries. Preserves attention.
5. **Verification markers.** ArsContexta's "verify" phase inspired `[unverified]`, `[needs-review]`, `TODO: verify with <role>`, `[stale]`.

## What we changed

1. **Target domain.** Personal knowledge management → codebase context. The interviewee is a pod lead, senior dev, QA — someone who holds the repo in their head.
2. **Input mix.** ArsContexta's inputs are user self-description. We additionally read `CLAUDE.md` / `AGENTS.md` / `README` as prior knowledge, mine recent PRs for historical context, and read source files to verify claims.
3. **Output structure.** ArsContexta uses `self/` + `notes/` + `ops/`. We use `hub.md` + `domains/` + `concepts/` + `invariants/` + `tasks/`. Invariants are promoted to first-class because they're the single biggest source of agent failures on unfamiliar code.
4. **Research-claim apparatus.** ArsContexta backs architectural choices with 249 research claims. V0.1 of `repograph` borrows the ideas without carrying the citation apparatus. Research grounding is a nice-to-have for a later version.
5. **Enhance mode is first-class.** ArsContexta is designed for one-user vaults. `repograph` is explicitly multi-contributor — a vault strengthens as more team members re-run the generator.
6. **Drift as a first-class concern.** `repograph-verify` exists because code rots faster than personal notes do. Content-hash anchoring, symbol-based citations, and a verify command are additions ArsContexta doesn't need.

## What we deliberately did not copy

- Queue-based orchestration and the `/ralph` processing pipeline. Not needed at V0.1 scale.
- Schema-enforcing hooks on every write. Adds friction; contributors can verify by running `/repograph-verify`.
- The full 6-Rs pipeline (Record / Reduce / Reflect / Reweave / Verify / Rethink). We use the ideas (especially Verify) without wiring them as distinct commands.

Credit where it's due: the underlying insight that *conversation + subagent isolation + atomic wiki-linked markdown* is the right substrate for Claude Code knowledge systems came from ArsContexta. `repograph` is a narrow application of that insight to codebases.
