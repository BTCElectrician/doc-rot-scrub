# Doc-Rot Pattern Catalog

Twelve patterns observed in real portfolio scrubs (2026), ordered
roughly by danger. Each entry: what it is, why it misleads agents, how to
detect it, disposition. "Field note" lines are real (anonymized) findings —
use them to calibrate how bad things can actually be.

## 1. Mirror docs (sibling-repo internals)

Docs whose primary subject is another repo/service: "how the frontend flow
works" living in the backend repo, function-app internals documented in the
ETL repo. They cannot stay true — nothing in this repo breaks when the
sibling changes.

Detect:
```
git ls-files '*.md' | grep -iE 'frontend|backend|azure|flow|interplay|<sibling-names>'
rg -il '<sibling repo/service names>' docs/
```
Then read candidates: is the *primary subject* the sibling? (Mentioning a
sibling is fine; narrating its internals is the offense.)

Disposition: RETIRE. Delete; replace citations with a link to the owning
repo, or a thin dated boundary contract (interface only, "last verified
<date>"). Boundary contracts that draw the ownership line WITHOUT narrating
sibling internals are the healthy replacement — keep those.

Ownership violations also run in a second direction: business, strategy, or
client-relationship content parked in a product repo when it structurally
belongs to a control-plane or client-docs layer. Detection is by *subject*,
not filename — compensation/pricing/negotiation topics, or a directory named
after the operator — compared by content against the owning layer's tree.
Same disposition; and when the content is confidential relative to who can
read the repo, FLAG that dimension explicitly (it outranks freshness).

Field note: 1,500-line "azure function flow" mirrors existed in three repos
at three different freshness levels; one was still labeled "Current Contract".
In a client's product repo, the operator's private pricing notes — literally
opening "Not for <client>" — sat inside the client's own codebase.

## 2. Replicated bundles across repos

The same doc bundle copy-pasted into several repos, each copy aging
differently. Four copies of "the truth" and no way to tell which lies.

Detect (multi-repo scrubs):
```
comm -12 <(git -C repoA ls-files '*.md' | xargs -n1 basename | sort -u) \
         <(git -C repoB ls-files '*.md' | xargs -n1 basename | sort -u)
```
Same distinctive directory names (`*-interplay/`, `*-flow/`) in ≥2 repos is
the signature.

Disposition: keep at most the copy in the owning repo (if fresh); delete the
rest per pattern 1.

## 3. Stale editor-agent rules (.cursor/rules, .cursorrules, Copilot instructions)

The sneakiest surface in the field. These files are auto-loaded by editors,
never listed in docs indexes, and nobody maintains them. They are read by
agents on every session.

Detect:
```
ls -d .cursor/rules/ .cursorrules .github/copilot-instructions.md \
      .github/instructions/ .claude/rules/ 2>/dev/null
```
then verify EVERY factual claim against the codebase: framework and version
(check the real package/dependency file), endpoints, model IDs, index/service
names, file paths. Also check the file still loads where its author expected:
Cursor treats a root `.cursorrules` as legacy and lets `.cursor/rules/*.mdc`
override it, and ignores plain `.md` files in `.cursor/rules/`. A rule that
no longer loads in its editor still misleads any agent that reads it.

Disposition: DELETE when generic-2024-era or wrong (point agents at the
repo's cross-tool surface, usually `AGENTS.md`, which current Cursor,
Copilot, and Codex releases all read); FIX in place only when the file is
genuinely used and mostly right.

Field note: one repo's rules claimed it was a Next.js app — it was Flask.
Another cited an endpoint renamed months earlier, dead model IDs, and paths
from a different machine (`/mnt/project/...`). Three of four repos had stale
rules; zero had accurate ones.

## 4. Byte-identical duplicate doc trees

A directory duplicated wholesale (usually live copy + `legacy/` copy) so
"current" and "archive" contain the same bytes and both rot together.

Detect:
```
diff -rq docs/<dir>/ docs/legacy/<dir>/
```

Disposition: DELETE the live copy, keep the archived one — but re-run
`diff -r` immediately before `git rm`; if any file diverged, `git mv` the
divergent file into the archive instead of deleting it.

Carve-out: two identical trees are NOT this pattern when the pair is
intentional — a live surface documents the sync relationship and a checker
script verifies it. That pair is KEEP-both; deleting one side breaks
tooling. The distinguishing test: does tooling point at the pair, or is one
side just an abandoned copy nothing checks?

Field note: three full directories (37 files) were byte-identical duplicates
of their already-archived twins. In another repo, an identical skills tree
was a deliberate synced pair guarded by `scripts/check_codex_skills.sh` —
deleting either side would have broken the check.

## 5. Hand-maintained context packets (the "workflow doc")

The founding artifact of the genre. Docs the operator personally kept
updated — extracted functions, file-path attributions, key logic — so
neither the human nor a small-context agent had to hold the whole codebase.
Updated every few days while the habit lasted; frozen at whatever week it
stopped. These are the most-trusted, fastest-rotting docs in a repo: they
contain real code excerpts that WERE true when copied, so they read as
maximally authoritative, and they name exact functions and files — precisely
the claims an agent acts on without checking.

Detect:
```
rg -l -U '```' docs/ *.md          # docs embedding code blocks
git log --format=%cs -- <file> | head -15   # update-cadence signature
```
The git signature is distinctive: a burst of commits every few days, then
silence. For each candidate, pull 2–3 of its named file paths and function
names and check them against the codebase — do the files exist, do the
functions still live there with those signatures?

Disposition: check the document's current consumer and whether its upkeep is
justified. Retire stale code narration when live code inspection meets that
need; keep a small verified map for a named consumer when it earns its upkeep.
Preserve unique rationale, decisions, constraints, or unfinished work in the
owning current surface before retiring a packet. Do not create a replacement
map by default.

Field note: operators describe maintaining these by hand "every two days" as
a workaround before agents could read a whole codebase and produce a
believable refactor. Abandoned copies later derailed capable agents, which
preferred the confident stale excerpt over reading the live source.

## 6. Weak-model-era briefing artifacts

Old implementation plans, "context for the AI" files, pasted chat diagnoses,
dated week-folders, one-off patch snippets. These may contain stale behavior
claims or unique decisions and unfinished work; inspect before disposition.

Detect: filename patterns `*PLAN*`, `*CONTEXT*`, `*GUIDE*`, `*PROMPT*`,
`*IMPLEMENTATION*`, `*HANDOFF*`, `week-*/`, month-named dirs; cross-check
age (`git log -1 --format=%cs -- <path>`) and whether any agent surface
references them (`rg -uu -l "$(basename <path>)"`).
Rule of thumb: >6 months untouched AND unreferenced from any surface.

Disposition: retire verified stale material; archive or link unique decisions,
constraints, requested deliverables, and unfinished work in the owner's
current surface. Age and filename alone do not justify deletion.

Field note: 75–81% of tracked markdown in two repos fell in this bucket,
including a 749-line pasted AI diagnosis.

## 7. Off-topic squatter content

Bulk-copied corpora that have nothing to do with the repo (research dumps,
training material) — usually one giant import commit, often with " copy" in
folder names.

Detect: directory md-counts wildly out of proportion to the repo's purpose;
`git log --diff-filter=A --format='%cs %s'` on the directory shows one bulk
commit.

Disposition: verify the content exists at its true origin
(`diff -r` against the origin repo/folder); if confirmed, `git rm` here; if
origin is missing or contents diverge, relocate first or leave and report.
Never delete a possibly-only copy.

Field note: a 140-file domain-research corpus was squatting in an unrelated
service repo; 139/140 files were byte-identical to their real home, so
deletion was safe — but only provably so after the diff.

## 8. Half-finished retirements

A previous cleanup added "do not trust this" banners or dropped docs from one
reading list, but other startup surfaces still cite the same files as
authoritative. Agents follow the mandated read order and hit the permissive
framing first.

Detect: for every doc with a banner or "historical" label, grep which
surfaces still cite it: `rg -uu -l "$(basename <doc>)" AGENTS.md CLAUDE.md README.md docs/`

Disposition: finish the retirement consistently. If deletion is supported by
the content and consumer checks, remove the doc and fix every citing surface
in the same change. Otherwise archive or correct the doc and its citations.

Field note: a repo's STATUS.md said "do not read the local mirrors" while its
AGENTS.md — read first — still said "the local mirror is <file>; trust it".

The same pattern occurs as incomplete coverage: a banner-adding pass that
caveats four of five sibling docs and misses the fifth, or a reading-list
purge applied in one surface but not another. Audit retirement passes as
sets — find the whole genre cluster, then check every member got the same
treatment.

## 9. Stale generated docs with weak regeneration

Generated reference docs (schema snapshots, API dumps) where the regen step
only warns on drift instead of failing, so the file silently falls behind.

Detect: find the generator (`rg -l '<filename>' scripts/ Makefile *.sh`);
check whether CI/build fails or merely warns on diff; spot-check the file
against present-day reality.

Disposition: REGENERATE via the real generator — never hand-edit, never
delete (build coupling). If regen needs live infrastructure you lack, flag as
known-stale in the PR body and move on.

Field note: a "current database schema snapshot" was missing the repo's
dominant feature of the previous five months because the build step only
warned on drift.

## 10. Search-hygiene drift (the repo-own ≠ live trap)

Repos with an archive policy (".rgignore keeps archival docs out of default
search") accumulate drift in both directions: stale docs outside the ignore
set, and — worse — cleanups that un-ignore archival docs because they are
"repo-owned". Ownership and freshness are different axes; visibility follows
freshness.

Detect: read the docs README for a stated archive policy; then compare
`rg --files docs/` (honors ignores) against `git ls-files docs/` and check
each exposed doc's age/promotion status.

Disposition: FIX the ignore file so unpromoted/stale docs stay hidden by
default while explicitly-promoted live docs stay visible (negation entries).
Verify from the repo root with no path argument — explicit paths bypass
ignores by design, which is the sanctioned escape hatch.

Field note: a scrub agent un-ignored an entire conceptual-flow directory
because the docs were "repo-own"; a PR review bot caught it. The fix restored
the blanket ignore plus one `!live-map.md` negation.

## 11. Orphaned self-referencing bundles

Doc clusters that only cite each other — zero inbound links from any agent
surface or index. Harmless-looking, but they surface in content greps and
read as authoritative once found.

Detect: for each cluster member, `rg -uu -l "$(basename <file>)"` — if all
hits are inside the cluster, it is orphaned.

Disposition: orphanhood is a linking signal, not a verdict. Check accuracy,
unique content, and unfinished work. Remove verified rot only after those
checks; an accurate orphan may reveal a linking gap or pending work — see
pattern 12. Correct any stale index pointers in the same change.

Field note: an eight-file mirror bundle sat quarantined in a legacy folder,
referenced by nothing but itself, for five months after everyone forgot it.

## 12. Lost-work docs (accurate, unexecuted, orphaned)

The inverse threat of every other pattern: a doc verified CORRECT — an ops
order, hardening mission list, triage plan — whose described work was never
finished, and whose pointer trail got severed, often by the very cleanup
that executed its first mission. Nothing here misleads an agent; instead,
real pending work (sometimes on money or security surfaces) becomes
invisible.

Detect: for docs shaped like mission lists (numbered tasks + done-criteria +
sequencing), grep for each mission's named artifacts — the test files,
modules, or scripts it says should exist. Executed or not? If partially
executed AND absent from the current status/next-steps surface, flag it
regardless of calendar age.

Disposition: FLAG for the operator — never DELETE on orphanhood alone. The
right fix is usually re-linking the remaining missions from the live status
surface, or explicitly declaring them dropped.

Field note: a verified-accurate hardening ops order sat orphaned with 3 of 4
missions unexecuted — including missing behavior tests on payment-webhook
and row-level-security surfaces. In another repo, an open hygiene issue
survived a month of commits that touched the exact file it flagged. "Known"
is not "done", and "accurate" is not "linked".

---

## Keep-list heuristics (what NOT to touch)

- Startup surfaces that pass spot-checks (`AGENTS.md`, `STATUS.md`,
  `docs/current/**`-style trees) — in the field these were ~97% accurate;
  the rot lives in the old strata, not the maintained handoff path.
- Boundary/ownership contracts between services (dated, interface-only).
- Archive gate READMEs and any archived file that a live surface explicitly
  points into ("see <archive>/incident-X.md").
- Decision records / ADRs — dated by genre; they claim to be history, not
  current truth, so they do not mislead.
- Anything a workflow, build script, generator, dashboard, or skill reads —
  the Phase 0 trip-wire list exists to find these.
- Prompt/template assets that are runtime inputs (not documentation at all).
