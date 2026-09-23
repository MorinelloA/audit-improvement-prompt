# Audit‑to‑A — A Reusable Deep‑Audit Prompt for Any Codebase

A portable, tool‑grounded audit methodology that grades a codebase against an **operational
rubric** (a deterministic weighted formula, not a vibe), records findings in a **persistent
ledger** so progress compounds across runs, and emits a **staged remediation plan** that — when
executed — provably reaches an "A." It works for one repo or many, in any language.

**The prompt itself lives in [prompt.md](prompt.md).** This README is the documentation for the
human; the prompt file is what you hand to the LLM.

> **Why this exists.** A normal "review my code" prompt produces a different ad‑hoc grade every
> time and re‑finds the same handful of salient issues. It has a structural ceiling: the grade is
> an LLM judgment call (anchors to "competent‑but‑imperfect" ≈ B forever), prior fixes are never
> credited, thoroughness is capped, findings are eyeballed instead of tool‑measured, and test/branch
> coverage is never quantified. This prompt removes that ceiling by making the grade a *formula over
> measurable gates* — once every gate is green, the grade **is** an A by construction, with no
> opinion left to disagree with.

---

## How to use

1. **Open the repo you want audited** in the agent's working directory. There is nothing to fill
   in: Section 0 of the prompt resolves every input from the checkout itself (see *What it decides
   for you* below). To override anything, say so in your message ("quick budget", "exclude
   `src/legacy`") or edit `audit/config.json` after the first run.
2. **Give the LLM the entire `prompt.md` file** (it is self‑contained, including the per‑stack
   census command cookbook in Appendix A, the sink/footgun grep list in Appendix B, and the
   one‑page sub‑agent briefs in Appendix C).
3. **Pick the execution mode:**
   - **Agentic LLM with tools** (Claude Code, Cursor, an SDK agent, anything with shell + file
     access): it runs the whole thing itself — census, finders, verification, writes the
     scorecard/ledger. This is the intended mode.
   - **Multi‑agent orchestration available** (e.g. Claude Code's Workflow tool, or any fan‑out
     harness): use it for Phase 1 (one agent per *repo × lens*) and Phase 3 (adversarial verifiers).
     Far more thorough. **Read Section 4.0 first** — verification is ~90% of the cost, so the free
     collapse/triage pass in Phase 2 has to happen before it, not after.
   - **Plain chat LLM, no tools:** it will print the exact census commands for *you* to run, you
     paste the outputs back, and it triages from there. Slower, but the method still holds.
4. **Re‑run after each remediation milestone.** The scorecard diff is the proof the grade is
   climbing. Same code state + same pinned SHAs ⇒ same letter (reproducible by design).

## What it decides for you

Every former input is auto‑resolved, written to `audit/config.json` with its source (`auto`,
`default`, or `human`), and printed in the scorecard header so the decision is reviewable rather
than silent. Precedence is: your message → an existing `config.json` → auto‑detection.

| Input | Default resolution |
|---|---|
| Repo set | Git toplevel of the working directory; an umbrella folder yields one `repo` per child repo; workspaces are modules, not repos |
| Stacks, project name | Manifests; `origin` remote basename |
| Audit workspace | `audit/` at the repo (or umbrella) root, added to `.git/info/exclude` so no tracked file changes |
| Out of scope | Generated code, vendored deps, build output, lockfiles (lint only), fixtures (lint only), migrations — **disclosed**, never assumed beyond that |
| Layer map | Each module classified `core` / `api` / `ui` / `client` / `test` / `infra` by convention, then dependency direction; coverage gates apply per layer |
| Critical flows | 3–5 flows from the entry‑point inventory (auth, primary write path, hottest read path) |
| Budget, DB policy, "A" threshold | `standard`, `review-only`, 3.85 |
| Write policy | Writes only under `audit/` plus uncommitted analyzer config; never commits, pushes, branches, or edits tracked source |

Dimensions or clauses that genuinely don't apply (no HTTP surface, no database, no UI) are marked N/A
with inventory evidence and drop out of the denominator. "Hard to measure" is not N/A; it fails closed.

## What makes it different

The five things that make this different from an ordinary audit — keep them intact if you adapt it:

| Ordinary audit | This method |
|---|---|
| Grade is an LLM vibe → anchors to B every run | **Grade is a weighted formula** over measurable per‑dimension gates |
| "Ignore prior reviews / fresh eyes" → can't compound | **Persistent scorecard + ledger**; credit closed items; progress accumulates |
| "Prefer 15 findings over 50" → caps thoroughness | **No finding cap; exhaustive module × dimension matrix; loop‑until‑dry** |
| Eyeballed by reading code | **Tool census is ground truth**; the model triages a complete machine inventory |
| "Tests exist → looks fine" | **Coverage (esp. branch) is a primary, quantified workstream**, plus assertion‑less, skipped, and mutation‑sampled tests |
| "Auth looks fine" from reading a few controllers | **Entry‑point inventory** — every route, consumer, job and webhook with its auth and validation binding — so authz and validation are measurable |
| "0 warnings" taken at face value | **Suppression census** and **known‑violation probes**: a clean result from a tool that scanned nothing, or a baseline built on `#pragma disable`, is a failed gate |

## Cost & resumability

Exhaustive discovery is cheap; *arguing about every finding at equal depth* is what makes a deep audit
expensive. The prompt separates the two, so thoroughness survives a budget instead of competing with it.

- **Verification is the budget.** It scales as *findings × skeptics*. Everything else — census,
  finders, synthesis — is a rounding error next to it.
- **Free filtering runs first.** Dedup, root‑cause collapse, and rubric mapping (Phase 2) are plain
  computation, and the grade‑relevance gate is one judgment by the orchestrator. None of it spawns an
  agent, and all of it runs *before* verification (Phase 3).
  Deduping after verifying means paying 3× to argue about findings you were about to merge or drop.
- **Tool‑emitted findings skip the panel.** If an analyzer, scanner, or coverage report asserts it, the
  tool *is* the verification. Adversarial skeptics exist for `inferred` behavioral claims — the only
  findings that can be plausible‑but‑wrong.
- **Depth follows consequence.** Skeptic count is set by severity, with a severity rubric that anchors
  finders (ungrounded finders over‑rate severity badly, which inflates both cost and remediation order).
- **Machines file tool findings.** Analyzer hits, vulnerable packages, uncovered files and
  suppressions become ledger candidates by script, grouped by rule and directory. No agent re‑types
  500 warnings into JSON; finders spend tokens only on root‑cause collapse and on what no tool emits.
- **Fan‑out matches the work.** Tool‑led lenses (dead code, duplication, dependencies, coverage) get one
  agent each with repos batched at every budget, because their input is a table. Judgment‑led lenses
  fan out per repo. Medium/Low findings in the same module share a skeptic call; Critical/High never do.
- **Later rounds are targeted, not repeated.** Round 1 fills a module × lens coverage matrix; a
  one‑agent completeness critic then seeds only empty cells, the neighbourhood of verified
  Critical/High findings, and mechanisms the ledger relies on that nobody read. A full matrix plus an
  empty critic is a stop condition in itself — cheaper and stronger than "two more rounds found nothing."
- **Sub‑agents get a one‑page brief**, not the whole prompt (Appendix C). The orchestrator is the only
  reader of Sections 0–7.
- **Nothing is paid for twice.** Findings hit `backlog.jsonl` as `candidate` the moment a finder
  returns; every verdict is appended to `verdicts.jsonl` keyed by a content‑derived `fingerprint`; each
  phase checkpoints `run-state.json`. A run that dies mid‑audit resumes owing only the unverified
  remainder — no re‑census, no re‑finding, no re‑verification.
- **Budget scales depth of argument, never breadth of search** — and the run states which stop
  condition ended it, because *"stopped: budget"* and *"stopped: matrix full, critic empty"* are
  different claims about completeness.

Integrity is unchanged and in places stricter: a skeptic that **crashed did not vote** (infrastructure
failure must never be recorded as "refuted"), severity corrections returned by verification are
**applied**, not just logged, and anything left `unverified` is named and counted in the scorecard
rather than silently dropped.

## Quick‑start checklist

- [ ] Open the target repo in the agent's working directory and hand it [prompt.md](prompt.md). Nothing to fill in.
- [ ] Phase 0: it resolves `audit/config.json`, pins SHAs (+ dirty‑tree digest), records any provenance warning, runs the census with a `manifest.json` proving each tool scanned something.
- [ ] Review the auto‑derived scope and layer map in the scorecard header; edit `config.json` if you disagree.
- [ ] Phase 1–5: scripted ingestion → finders → collapse/triage → adversarial verify → targeted rounds → synthesis.
- [ ] Read the output contract (prompt Section 6), especially the residual line: stop condition, unverified count, failed or N/A census steps, cost.
- [ ] Pick the top milestone task by grade leverage; execute; re‑run the audit; watch the scorecard diff.
