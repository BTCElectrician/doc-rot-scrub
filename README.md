# doc-rot-scrub

[![Validate skill](https://github.com/BTCElectrician/doc-rot-scrub/actions/workflows/validate.yml/badge.svg)](https://github.com/BTCElectrician/doc-rot-scrub/actions/workflows/validate.yml)

An agent skill that audits a repository's markdown for stale docs that
mislead coding agents, then cleans them up without losing anything that
still matters. It is written for Claude Code and Codex. Because it uses the
open [Agent Skills](https://agentskills.io/specification) format
(`SKILL.md`), Cursor and GitHub Copilot can load it as well, though it has
not been exercised there.

## The problem

If you have been coding with AI assistants since 2023, your repos probably
hold markdown you never meant to keep: hand-maintained "workflow docs" that
list key functions and file paths, old PRDs, pasted AI diagnoses, write-ups
of how some other repo works. Keeping those docs current was a reasonable
habit when models had small context windows and could not read a whole
codebase. Most people dropped the habit and left the docs in place.

Current agents read your code directly, but they still treat repo markdown
as trustworthy context. Code fails loudly when it drifts. Markdown never
does. A confident two-year-old doc is cheaper for an agent to believe than
re-deriving the truth from source, so it sends the agent off course before
anything contradicts it. Even an accurate old doc costs context and anchors
the agent's plan on an outdated map of the code.

## What it catches

The [pattern catalog](references/rot-patterns.md) has 12 patterns, each
found in real repository audits and each with detection commands and a
disposition rule:

1. Mirror docs: one repo narrating another repo's internals.
2. Replicated bundles: the same doc set copied into several repos, each
   copy aging differently.
3. Stale editor rules: `.cursor/rules/`, `.cursorrules`, and Copilot
   instruction files that load every session and that nobody audits.
4. Byte-identical duplicate doc trees (a live copy and its archive twin).
5. Hand-maintained context packets, the "workflow doc" described above.
6. Old plans, PRDs, and pasted AI diagnoses.
7. Unrelated corpora bulk-copied into a repo.
8. Half-finished retirements, where a cleanup bannered some docs but a
   startup file still cites them as current.
9. Generated docs whose regeneration step only warns on drift.
10. Search-ignore files that hide or expose the wrong docs.
11. Orphaned bundles that only cite each other.
12. Lost-work docs: accurate plans whose remaining work fell off the status
    page.

## How it works

The skill classifies each doc on two independent axes. **Ownership** asks
whether this repo owns the behavior the doc describes. **Freshness** asks
whether the doc is current or a superseded snapshot. A doc can belong to
the repo and still be stale, or be accurate and still be another repo's to
maintain, so the two are judged separately.

The work runs in phases:

0. **Recon.** Find out whether pushing deploys, whether any build, script,
   or runtime code reads files under `docs/`, and whether another agent is
   working in the same checkout.
1. **Read-only audit.** Start with the files agents load at startup
   (`AGENTS.md`, `CLAUDE.md`, editor rules), since a stale doc cited from
   one of those does the most damage. Set aside runtime data and end-user
   docs, which are out of scope. Spot-check concrete claims (paths, routes,
   commands) against the code. The output is a keep / archive / remove /
   unresolved list with evidence for each item.
2. **Decision.** Destructive changes the evidence does not settle, and
   anything outside what you have authorized, come to you. If you already
   approved a specific cleanup and the evidence supports it, the agent
   proceeds. Anything unresolved stays as it is.
3. **Execution.** A direct commit, a branch and PR, or an isolated git
   worktree, depending on what recon found. Links to removed files are
   fixed in the same change.
4. **Prevention.** A guardrail or rule is added only when the audit found a
   concrete way for the rot to come back.

[`SKILL.md`](SKILL.md) is the file the agent reads; it has the full
procedure.

## Safety model

The agent is instructed to:

- audit read-only before changing anything;
- stay out of a checkout that another agent or person is actively using,
  and work in a separate git worktree instead;
- confirm with `diff -r` that bulk-copied content exists at its origin
  before deleting it, and leave it in place if that cannot be confirmed;
- regenerate generated docs rather than hand-editing or deleting them;
- leave runtime data and end-user documentation alone;
- skip any file that appears to contain a credential, and never print it.

These are instructions, not code-enforced guarantees. Review the diff as you
would any other agent change.

## Example

The audit output looks like this. Paths and findings below are illustrative,
modeled on patterns in the catalog.

```
REMOVE     .cursor/rules/project.mdc
           Says "Next.js app"; the repo is Flask (pyproject.toml). Loaded
           every session.
REMOVE     docs/frontend-flow.md
           Mirror doc narrating the web repo's internals. AGENTS.md cites it;
           that pointer becomes a link to the owning repo in the same change.
ARCHIVE    docs/2024-auth-plan.md
           Superseded by the current auth code, but it is the only record of
           why sessions moved server-side. Banner plus pointer to the owner.
FLAG       docs/hardening-orders.md
           Accurate. Three of four tasks were never done and nothing current
           links to it. Your call: re-link or drop.
REGENERATE docs/schema.md
           Generated by scripts/dump_schema.py, but the build only warns on
           drift and the file is months behind. Rerun the generator; never
           hand-edit.
```

## Install

**Claude Code**

```bash
git clone https://github.com/BTCElectrician/doc-rot-scrub.git ~/.claude/skills/doc-rot-scrub
```

For a single project, clone into `.claude/skills/doc-rot-scrub` inside that
repo instead. Claude Code picks up new skills without a restart; if
`~/.claude/skills` did not exist when the session started, run
`/reload-skills`.

**Codex, Cursor, GitHub Copilot**

All three read the shared Agent Skills location:

```bash
git clone https://github.com/BTCElectrician/doc-rot-scrub.git ~/.agents/skills/doc-rot-scrub
```

For a single repo, use `.agents/skills/doc-rot-scrub`. Codex still reads the
older `~/.codex/skills`, but that location is deprecated. If Codex does not
list the skill, restart it.

To update later, run `git pull` in the directory you cloned into.

## Use

Describe the problem ("old docs keep sending the agent off course", "clean
up the markdown in this repo") and the skill should trigger on its own. To
call it explicitly:

- Claude Code: `/doc-rot-scrub`
- Codex CLI or IDE extension: mention `$doc-rot-scrub` in your prompt
- Codex, non-interactive:

  ```bash
  codex exec 'Use $doc-rot-scrub to audit this repo'
  ```

  Keep the single quotes so your shell does not expand `$doc`. `codex exec`
  runs in a read-only sandbox by default, which is enough for the audit.
  Add `--sandbox workspace-write` when you want it to apply approved
  changes.

## Requirements

git and a shell. The detection commands also use `rg` (ripgrep), `diff`,
and `comm`. The skill ships no scripts of its own; the agent runs these
commands directly.

## Related: Claude Code's `/doctor prompt-audit`

Claude Code has a built-in `/doctor prompt-audit` that checks `CLAUDE.md`,
`AGENTS.md`, and the rules, skills, and commands under `.claude/` for
outdated or conflicting instructions. Use it for those files. doc-rot-scrub
covers the rest of the markdown tree that agents end up reading (old plans,
mirror docs, duplicate trees, other editors' rule files), uses git history
as evidence, and also runs in Codex.

## Repository layout

| Path | Purpose |
| --- | --- |
| `SKILL.md` | Instructions the agent loads when the skill triggers |
| `references/rot-patterns.md` | Pattern catalog with detection commands, read on demand |
| `.github/workflows/validate.yml` | CI check against the Agent Skills spec |

## Contributing

The catalog comes from audits of one developer's repositories, built solo
between 2023 and 2026. It is probably missing patterns common in team
repos, monorepos, and wiki or Notion exports. If an audit turns up rot that
does not fit the catalog, or a step in `SKILL.md` was ambiguous, open an
issue or a PR. A pattern earns a catalog slot when it is recurring (seen in
at least two repos), dangerous (an agent would act on it), detectable (a
command finds it), and has a disposition distinct from the existing ones.

To validate locally before opening a PR (from a checkout whose directory is
named `doc-rot-scrub`, since the spec requires the name to match):

```bash
uvx --from 'git+https://github.com/agentskills/agentskills#subdirectory=skills-ref' skills-ref validate "$PWD"
```

## License

MIT. See [LICENSE](LICENSE).
