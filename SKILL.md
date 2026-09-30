---
name: doc-rot-scrub
description: Audit and scrub stale AI-era markdown so it stops misleading coding agents — hand-maintained "workflow"/context docs full of extracted functions and file maps, old PRDs and plans, cross-repo "mirror docs", duplicate doc trees, and stale editor rules (.cursor/rules, .cursorrules, copilot-instructions). Use whenever the user mentions old docs or markdown misleading agents or polluting context, an agent going off course from a stale doc, documentation debt, doc cleanup/audit/scrub, mirror docs, old PRDs/context files they used to keep updated by hand, or a repo full of markdown from the early vibe-coding era — even if they never say the word "scrub".
---

# Doc-Rot Scrub

## Purpose

Check whether agent-facing documents are accurate, who consumes them, and
whether they still help the current work. Verify implementation claims in live
code and deployment claims at runtime. Preserve requested deliverables,
non-code decisions and constraints, unique evidence, and unfinished work.
Do not infer obsolescence from age or model capability. A small verified map
may stay when a named consumer justifies its upkeep; default against new prose
that merely narrates code.

## The two axes (the core doctrine)

Classify every doc on two independent axes. Do not collapse them.

1. **Ownership** — does this repo own the behavior the doc describes?
   A doc whose primary subject is another repo/service's internals is a
   **mirror doc**. Mirrors cannot stay true: nothing breaks here when the
   other repo changes.
2. **Freshness** — is this doc live-and-promoted, or a superseded snapshot?
   A doc can be fully repo-own and still be stale ("repo-own ≠ live").

Dispositions by quadrant:

| | Fresh | Stale |
|---|---|---|
| **Repo-own** | KEEP when it serves a named need | DEMOTE, ARCHIVE, or REMOVE after checking unique content and consumers |
| **Mirror** | RETIRE → replace with an owner link or a thin, dated boundary contract | REMOVE after checking unique content and consumers |

Archive-with-banner only when a doc holds unique rationale captured nowhere
else. Banner format: date, "ARCHIVED SNAPSHOT — do not treat as current",
pointer to the owning source of truth.

## Operating rules (non-negotiable safety)

- **Inspect before disposition.** Audit read-only first and present the
  proposed changes. Execute within existing operator authorization; seek a
  decision only for uncertain destructive dispositions or an action outside
  that authorization. A push-triggered release follows the owner's release
  rule.
- **Never touch a working tree another agent (or the user) is actively
  using.** Signs of life: dirty files, fresh mtimes, HEAD advancing between
  your looks. If active, do all writes in a `git worktree` cut from
  `origin/<default>` on a new branch, merge remote-side via PR, and state the
  expected conflicts in the PR body.
- **Never destroy a possibly-only copy.** Before deleting bulk-copied content
  (research dumps, corpora), verify it exists at its origin (`diff -r`). If
  origin is missing or ambiguous, leave it and report.
- **Code establishes actual behavior.** Verify implementation claims against
  code and runtime claims at the runtime boundary. A user requirement or
  approved decision may describe desired behavior that code has not reached;
  preserve that gap instead of declaring the requirement wrong.

## Phase 0 — Trip-wire recon (before anything else)

Answer these for each target repo; they decide execution mechanics later:

1. Does pushing the default branch deploy? Check workflows AND prose —
   platform-native integrations (Render, Vercel, Netlify) are invisible in
   `.github/workflows/`; READMEs and agent files usually state them.
2. Do workflows/scripts read or publish docs? Look for: Pages/site builds fed
   from files under `docs/`, guardrail scripts with hardcoded doc-path
   allowlists, build scripts that regenerate doc files, CI path filters on
   `docs/**`. Also grep application source for import/require of paths under
   `docs/` — a runtime file living in a docs tree is load-bearing at
   compile/deploy time, and invisible to any markdown-only sweep.
3. Do deploy-exclude files (`.funcignore`, `.vercelignore`, package globs)
   already exclude docs? This narrows one deploy path; check remaining build,
   automation, and runtime consumers before judging risk.
4. Is there a search-hygiene contract (`.rgignore`, archive policy in a docs
   README)? You must preserve it — see pattern 10 in the reference.
5. Does automation consume specific docs (autofix workflows reading a prompt
   file, skills reading an asset map)? Those files are load-bearing.
6. Is another agent active in the checkout (rule above)?

## Phase 1 — Audit (read-only)

Work in this order; the order is the method:

1. **Agent surfaces first.** Enumerate every file that instructs agents:
   `AGENTS.md`, `CLAUDE.md`, `README` "start here" sections, docs-index
   READMEs, `.cursor/rules/*` and `.cursorrules`,
   `.github/copilot-instructions.md`, `.codex/` prompts/skills. Extract each
   surface's mandated reading list. **The danger is not a stale doc existing;
   it is a stale doc being cited from a startup surface.** Priority =
   referenced-from-surface × stale.
2. **Genre gate.** Before classifying anything, sort the inventory by
   intended reader — the freshness contract this skill enforces applies only
   to agent-facing documentation:
   - **Agent-facing docs** — in scope; everything below applies to these.
   - **Runtime data** — corpora, prompt/template assets that pipelines or
     automation consume, generated outputs, product content. Out of scope;
     record under DO-NOT-TOUCH. When you conclude a repo has zero
     runtime-data markdown, state the evidence (which corpus-adjacent
     directories you checked), not just the absence.
   - **End-user / external-facing material** — client manuals ("you"-voice,
     addressed to a named human), GitHub-rendered pages with badges. Zero
     citations from agent surfaces is *correct* for these, not a rot signal;
     default KEEP.
   - **Operator-private notes** (pricing, negotiation, compensation) parked
     in a shared or client-visible repo — FLAG with the confidentiality
     dimension named explicitly; relocation is an operator call, not a
     freshness disposition.
   In corpus-heavy repos runtime data can be a third to half of all markdown,
   and mis-auditing it as rot is the worst failure this skill can commit.
3. **Inventory.** `git ls-files '*.md'` (plus `.mdc` editor rules), count by
   directory, capture last-commit date per candidate
   (`git log -1 --format=%cs -- <path>`).
4. **Classify** against the two axes using the pattern catalog — read
   `references/rot-patterns.md` for the 12 field patterns with detection
   commands, dispositions, and real-world field notes. Staleness has two
   distinct signals; weigh both: **calendar staleness** (>6 months untouched
   AND unreferenced from any surface) and **supersession** (contradicted or
   replaced by a later doc or strategic pivot, regardless of age — a
   4-month-old doc describing an abandoned workflow is stale). A superseded
   doc still cited from a live surface is a FIX, not a silent delete; when
   supersession cannot be confirmed from inside the repo, FLAG it with the
   open question instead of guessing.
5. **Prior self-triage.** If the repo already contains its own hygiene or
   cleanup doc, treat it as a strong prior AND as evidence: verify its
   claims, note whether it is itself stale, and fold its unfinished items
   into the findings — an open "fix STATUS.md" item that survived a month of
   commits is itself a finding. Do not defer to it blindly, and do not
   re-derive from scratch as if it did not exist.
6. **Spot-check verification.** For the docs that surfaces DO mandate, verify
   2–3 concrete claims each against code: routes, file paths, commands, make
   targets, framework/model names. Report only failures. This is cheap and
   high-signal — it separates "old but true" from "confidently wrong".
7. For multi-repo scrubs: detect replicated bundles — the same
   directory/filenames appearing in several repos at different freshness
   levels is the classic mirror signature (compare `git ls-files` basenames
   across repos).

### Audit output

Give a concise keep/archive/remove/unresolved disposition list, with the
evidence and affected paths needed to judge each change. Add counts, dates,
consumer checks, or execution mechanics only when they affect a disposition or
the requested deliverable. Protect unique rationale, issued evidence,
unfinished work, runtime data, and automation-consumed files. Use a saved
manifest only when requested or needed for a substantial handoff.

## Phase 2 — Disposition decision

Present uncertain destructive choices for operator review. If the operator
already authorized the specific cleanup and the evidence supports it, proceed
without a repeated permission gate. Keep unresolved items intact.

## Phase 3 — Execution

Pick mechanics per repo from Phase 0 answers:

| Situation | Mechanics |
|---|---|
| No deploy on push, no active agent | direct commit to default branch |
| Deploy on push (main = production) | branch + PR when owner rules require it; apply existing release authorization |
| Active agent / dirty checkout | isolated worktree from verified remote base + PR when needed; inspect and resolve conflicts against current work |

Execution rules, learned the hard way:

- Fix pointer surfaces **in the same change** as the deletions. Deleting a
  mirror while `AGENTS.md` still says "trust the local mirror" leaves agents
  chasing dead links — worse than before. Grep every removed path across the
  repo (`rg -uu` to bypass ignore files) and fix danglers; replace citations
  with owning-repo links.
- `diff -r` duplicate directories against their archive twins immediately
  before `git rm`; if any file differs, move it instead.
- Do not hand-edit or delete generated docs with build coupling — rerun the
  generator, or flag as known-stale if it needs live infrastructure.
- Update guardrail allowlists / ignore files in the same change; when
  deciding search visibility, judge on **freshness**, not ownership
  (keeping a repo-own doc in the tree ≠ surfacing it in default search).
- Run the repo's own checks (its doc-guardrail scripts, linters, test suite)
  before commit; after opening PRs, read bot-review comments before merging —
  in field use a review bot caught a real regression a human missed.
- Never print anything that looks like a credential; if a file slated for
  deletion appears to contain a live secret, skip it and report.

## Phase 4 — Prevention (so it doesn't regrow)

Only change prevention rules or guardrails when the audit found a concrete
recurrence risk and that change is in scope. Do not add new docs, maps, or CI
policy by default.

## Prior audit context

In one four-repo audit, stale material concentrated in old strata and editor
rules while actively maintained startup surfaces were mostly accurate. This is
historical field evidence, not a deletion target or a prediction for another
repository. Check the current repository's consumers and authority links.

## Improving this skill

When a recurring gap affects a real disposition, consider updating the pattern
catalog. Promote a new pattern into `references/rot-patterns.md` only if it
is recurring (seen in ≥2 repos), dangerous (agents would act on it),
detectable (a command finds it), and has a disposition distinct from existing
patterns. Fix recurring instruction gaps at the point of ambiguity. Keep the
catalog sharp: a pattern that stops earning its slot gets merged or cut.
