# doc-rot-scrub

A Claude/Codex skill that finds and safely removes stale AI-era markdown
before it misleads your coding agent.

## The problem

If you've been "vibe coding" with AI since 2023, you probably have a pile of
markdown you didn't mean to keep: hand-maintained "workflow docs" listing key
functions and file paths (updated every couple of days so a small-context
model didn't have to read your whole codebase), old PRDs, pasted AI
diagnoses, "how the other repo works" write-ups. That habit was correct at
the time — those docs were load-bearing context for models that couldn't
hold a codebase in their head.

Modern agents read your code directly. But they still treat repo-local
markdown as pre-verified context — code fails loudly when it drifts, markdown
never does. A confidently-wrong two-year-old doc is cheaper for an agent to
trust than re-deriving the truth from source, so it steers the agent off
course before anything contradicts it. Even *accurate* old docs do damage:
they're pure noise in the context window, anchoring the agent's plan on a
stale map of your codebase.

This skill finds that rot, classifies it, and scrubs it — safely.

## What it catches

12 field-tested patterns, from real audits — not theoretical:

- **Mirror docs** — one repo narrating another repo's internals (or a
  product repo holding content that belongs in a different layer entirely)
- **Hand-maintained context packets** — the workflow-doc habit described
  above, detectable by its own git commit cadence
- **Stale editor rules** — `.cursor/rules/`, `.cursorrules`,
  `copilot-instructions.md` — auto-loaded every session, never audited
- **Half-finished retirements** — a cleanup that banners 4 of 5 docs in a
  set, or updates one surface but not the one an agent reads first
- **Lost-work docs** — the inverse problem: a verified-*accurate* doc whose
  work was never finished, orphaned by the very cleanup that did step one
- Byte-identical duplicate trees, off-topic squatter corpora, orphaned
  self-referencing bundles, stale generated docs, search-hygiene drift, and
  more — see [`references/rot-patterns.md`](references/rot-patterns.md) for
  the full catalog with detection commands and disposition rules.

## How it works

Every doc gets classified on two independent axes — **ownership** (does this
repo own the behavior described?) and **freshness** (is it current, or a
superseded snapshot?) — because a doc can be entirely repo-own and still be
stale, or entirely accurate and still be someone else's to document.

Then a phased pipeline:

1. **Trip-wire recon** — does pushing this repo deploy? Do any scripts read
   or publish docs? Is another agent actively working the same tree?
2. **Read-only audit** — agent surfaces first (the docs actually cited from
   `AGENTS.md`/`CLAUDE.md` matter most), then the rot-pattern sweep,
   producing one disposition manifest.
3. **A hard approval gate.** Nothing gets deleted until a human signs off on
   the manifest.
4. **Execution**, matched to risk — direct commit, branch+PR, or an isolated
   git worktree if another agent is live in that repo.
5. **Prevention** — a short rule added to `AGENTS.md` so the rot doesn't
   regrow.

Full detail is in [`SKILL.md`](SKILL.md) — that's the file your agent
actually reads.

## Install

**Claude Code** — drop the folder into your skills directory:
```bash
git clone https://github.com/BTCElectrician/doc-rot-scrub.git ~/.claude/skills/doc-rot-scrub
```
(or `.claude/skills/doc-rot-scrub` inside a single project for project-local use)

**Codex CLI / VS Code extension:**
```bash
git clone https://github.com/BTCElectrician/doc-rot-scrub.git ~/.codex/skills/doc-rot-scrub
# or .codex/skills/doc-rot-scrub at the root of one repo
```
Restart the CLI or extension if the skill doesn't show up right away.

## Use

Just describe the problem — old docs, an agent going off-course, "doc
cleanup" — and the skill triggers. Or invoke it directly:

```
Use doc-rot-scrub on this repo.
```

```bash
codex exec --full-auto "Use $doc-rot-scrub on this repo"
```

It stops at the approval gate and shows you exactly what it wants to
delete, move, or fix before touching anything.

## Contributing

The skill is designed to improve from real use. Every audit manifest
includes `NEW-PATTERN` and `SKILL-GAP` lines — rot that doesn't fit the
existing catalog, or places the instructions were unclear. If your repo
surfaces one, open a PR against [`references/rot-patterns.md`](references/rot-patterns.md)
or [`SKILL.md`](SKILL.md) with what you found. A pattern earns a catalog
slot when it's recurring (seen more than once), dangerous (an agent would
actually act on it), detectable (a command finds it), and has a disposition
distinct from what's already there.

The catalog so far comes from one operator's portfolio — a solo,
2023–2026 vibe-coding arc. It's almost certainly missing patterns specific
to team repos, monorepos, and wiki/Notion-export cultures. That's the gap
this project most wants filled.

## License

MIT — see [LICENSE](LICENSE).
