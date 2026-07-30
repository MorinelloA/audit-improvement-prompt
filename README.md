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

1. **Fill in the placeholders** in *Section 0 — Inputs* of [prompt.md](prompt.md) (`{{LIKE_THIS}}`).
   The only mandatory one is the repo path(s). Everything else has a sensible default.
2. **Give the LLM the entire `prompt.md` file** (it is self‑contained, including the per‑stack
   census command cookbook in its Appendix A).
3. **Pick the execution mode:**
   - **Agentic LLM with tools** (Claude Code, Cursor, an SDK agent, anything with shell + file
     access): it runs the whole thing itself — census, finders, verification, writes the
     scorecard/ledger. This is the intended mode.
   - **Multi‑agent orchestration available** (e.g. Claude Code's Workflow tool, or any fan‑out
     harness): use it for Phase 1/2 — one agent per *repo × lens*, then adversarial verifiers. Far
     more thorough.
   - **Plain chat LLM, no tools:** it will print the exact census commands for *you* to run, you
     paste the outputs back, and it triages from there. Slower, but the method still holds.
4. **Re‑run after each remediation milestone.** The scorecard diff is the proof the grade is
   climbing. Same code state + same pinned SHAs ⇒ same letter (reproducible by design).

## What makes it different

The five things that make this different from an ordinary audit — keep them intact if you adapt it:

| Ordinary audit | This method |
|---|---|
| Grade is an LLM vibe → anchors to B every run | **Grade is a weighted formula** over measurable per‑dimension gates |
| "Ignore prior reviews / fresh eyes" → can't compound | **Persistent scorecard + ledger**; credit closed items; progress accumulates |
| "Prefer 15 findings over 50" → caps thoroughness | **No finding cap; exhaustive module × dimension matrix; loop‑until‑dry** |
| Eyeballed by reading code | **Tool census is ground truth**; the model triages a complete machine inventory |
| "Tests exist → looks fine" | **Coverage (esp. branch) is a primary, quantified workstream** |

## Quick‑start checklist

- [ ] Fill Section 0 inputs in [prompt.md](prompt.md) (at minimum, the repo path(s)).
- [ ] Phase 0: pin SHAs, confirm you're on the canonical (not a stale) checkout, run the census.
- [ ] Stand up `audit/` (scorecard + empty `backlog.jsonl`).
- [ ] Phase 1–5: finders → adversarial verify → dedup/map → loop‑until‑dry → synthesis.
- [ ] Emit the output contract (prompt Section 6).
- [ ] Pick the top milestone task by grade leverage; execute; re‑run the audit; watch the scorecard diff.
