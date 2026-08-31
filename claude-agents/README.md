# Portable Claude Code Review & Planning Agents

A project-agnostic set of nine Claude Code subagents for C/C++ (and polyglot) code review,
planning, and refactoring. These were de-coupled from a specific codebase so they derive all
project specifics (conventions, target platform, error scheme, attack surface) from the target
repo's own docs at runtime rather than hardcoding them.

Drop them in `~/.claude/agents/` (user scope, all projects) or a project's `.claude/agents/`
(project scope).

## The agents

| Agent | Lens |
|---|---|
| `architecture-reviewer` | Macro structure, coupling, layering, threading model |
| `correctness-reviewer` | Logic bugs: state-ordering, off-by-one, contract-vs-impl, invariants, error paths |
| `security-auditor` | Memory corruption, injection, protocol/attack-surface, crypto, privilege |
| `embedded-efficiency-reviewer` | Resource/perf on embedded targets (Jetson/ESP32/RPi/x86 profiles built in) |
| `coding-standards-auditor` | Compliance with **the project's own** documented style; phased remediation |
| `reuse-pattern-reviewer` | Extract-when-it-helps refactoring (not blind dedup) |
| `master-code-reviewer` | Polyglot all-in-one reviewer (memory/efficiency/scope/correctness/UI/security) |
| `master-plan-reviewer` | Deep design/plan review: SOTA research, gap analysis, phased rollout |
| `ui-design-architect` | UI/UX design + accessibility review (only fires when UI is present) |

The first four make a solid parallel review set for any embedded C/C++ change; add the others
as the task calls for them.

## How they stay project-agnostic

Each agent reads the target project's context docs (`CLAUDE.md`, a style guide,
`ARCHITECTURE.md`, `.clang-format`, etc.) and treats those as ground truth. So the same
`coding-standards-auditor` audits GPL/3-space/`SUCCESS/FAILURE` in one repo and MIT/4-space/
`-errno` in another, without edits — as long as the repo documents its conventions. **Put your
conventions in the target repo's `CLAUDE.md`/style guide; don't bake them into the agents.**

## Per-setup knobs to check before reuse

These are Claude Code frontmatter/feature assumptions — verify they fit your environment:

- **`model:` frontmatter** — `master-code-reviewer` and `master-plan-reviewer` pin `model: fable`.
  Change or remove if your setup doesn't have that model alias (falls back to the session model).
- **`memory: user` frontmatter** (the three "master"/reuse agents) — enables persistent agent
  memory. Their bodies instruct writing to `.claude/agent-memory/<name>/` **resolved under the
  home directory** (the file tools don't expand `~`/`$HOME`, so the agent resolves an absolute
  path itself). No hardcoded username — portable as written.
- **`tools:` frontmatter** (`reuse-pattern-reviewer`) restricts the toolset; widen/narrow to taste.
- **Description trigger examples** mention generic protocols (MQTT, PCM, WebUI) purely as
  illustrations of *when to invoke* — harmless, edit only if you want domain-matched examples.

## License / provenance

Adapted from a personal agent set; no proprietary project content remains. Reuse freely.
