# Audit-to-A

Ask a model to grade a codebase and odds are it says B. Fix half of what it found, ask again:
another B, for a different handful of reasons. The grade is a judgment call, and nothing carries
over from one run to the next.

Audit-to-A turns the grade into arithmetic. Before judging anything, the agent runs real tools over
the code: linters, coverage, secret and dependency scanners, a dump of every route with its auth.
Findings go into a ledger that carries over between runs, so fixes get credit. The grade is ten
weighted dimensions scored against fixed gates; once every gate passes, the codebase is an A and
there's nothing left to argue about. Any language, one repo or a folder full of them.

[prompt.md](prompt.md) is what you give the model. This README is for you.

## Running it

1. Start your agent in the repo you want audited (or in a folder of repos), with the setup below.
   There's nothing to fill in; it reads what it needs off the checkout.
2. Give it all of `prompt.md`, appendices included.
3. Answer its one round of questions. Once it has looked at the checkout, it asks which missing
   scanners it may install, what only you can provide (a `gh` sign-in, a database the tests need),
   and whether its scope, layer map and budget look right. Then it runs to the end. Your
   answers go in `audit/config.json`; edit that file to change them, and the next run picks it up.
4. Work through the fix plan. It's split into milestones, M0 to M-A, with tasks ranked by how much
   they move the grade. Re-run the audit after each milestone and compare scorecards.

It's built for an agent with a shell and file access (Claude Code, Cursor and the like), which runs
every step itself and checks in after each phase. Expect a full run to take a while. If your harness
can fan out sub-agents, the prompt puts them to work as finders and skeptics; read Section 4.0
first, because that's where the cost gets decided. A chat model with no tools works too. It prints
the commands, you run them and paste back the output. Slower, same method.

You can also steer it from your message ("quick budget", "exclude `src/legacy`"). Your message wins
over `config.json`, which wins over what it works out for itself. The budget defaults to `standard`.
Database changes stay review-only unless you say "apply DB changes".

## Setup

Last checked October 2026.

| | Claude Code | OpenAI Codex |
|---|---|---|
| Model | Opus 5.5 (Fable 5.1 for an `exhaustive` budget) | GPT-6.1 Sol |
| Effort | `xhigh` | Extra High |
| Long run | Auto mode | `/permissions`, then Approve for me |

```bash
claude --model opus --effort xhigh --permission-mode auto --disallowedTools "Bash(git push *)" "Bash(git commit *)"
```

```bash
codex -m gpt-6.1-sol -c model_reasoning_effort=xhigh -c sandbox_workspace_write.network_access=true
```

The audit never commits or pushes, and the Claude command blocks both, which auto mode would
otherwise allow. Codex's sandbox keeps the network off unless you turn it on, as the command does. In
a desktop app, make the same choices from its menus. Where the harness lets it pick a model per
sub-agent, as Claude Code does, the prompt runs the cheap roles (tool-led finders, Medium and Low
skeptics) one tier down. A full run outgrows Claude Pro and ChatGPT Plus limits; Claude Max, ChatGPT
Pro or an API key suits it.

## What you need

- A checkout whose dependencies install and whose tests run. Coverage is measured by running them.
- The scanners for your stacks (the prompt's Appendix A lists them), or your go-ahead to install
  them. A tool that can't run fails its gates closed.
- `gh` signed in (or `glab`, `az`) for the branch-protection check. Without it, that check is
  recorded as unverifiable.
- Any services the tests need, such as a database container.
- Network access, for the vulnerability and end-of-life lookups.

## What it writes

```text
audit/
  config.json      the inputs it worked out, and where each came from
  scorecard.md     the grade and the evidence for it, plus run history
  backlog.jsonl    every finding ever filed (fixed ones stay, marked closed)
  verdicts.jsonl   each skeptic's verdict, so nothing gets argued over twice
  run-state.json   a checkpoint, so a dead run can resume
  closed.md        fixes with their commits, and regressions
  census/<date>/   raw tool output, plus a manifest of what each tool scanned
```

`audit/` goes in `.git/info/exclude`, so it stays out of `git status`. The census may add analyzer
settings to your build config; those stay uncommitted, and the scorecard lists them. The audit never
commits, pushes, creates a branch or edits your code. Fixing things is a separate request.

## How the grade works

Ten dimensions, weighted: security (18), testing (18), architecture (15), code quality (13),
correctness (12), performance (8), dependencies (6), DevEx and ops (6), accessibility (5, UI only)
and documentation (4). Each scores 0 to 4 on whether its gates pass. The weighted average is a GPA,
and 3.85 or better is an A. A script does the math (decimal, rounded half up), and the scorecard
pins every repo's commit, so the same code should always get the same letter.

Gates fail closed. A check that applies but can't be measured counts as a miss, and N/A has to be
backed by the inventory: no endpoints, no endpoint-auth check. Clean results get checked too. The
census records how many files each tool scanned and, where it's cheap, plants a known problem in
`audit/probes/` (a fake AWS key, say) to make sure the tool flags it. Zero warnings from a scanner
that read zero files fails the gate. So does a zero propped up by `#pragma warning disable`.

## Cost

Finding problems is cheap. The bill comes from arguing about them: findings no tool proved go to
skeptic agents told to refute them, and 270 unfiltered findings at three skeptics apiece is about
810 agents. So the free work runs first. Duplicates merge and repeats collapse to one root cause (35
untested pages missing the same test seam become one finding) before any skeptic is spawned.
Anything a tool already proved skips the panel, and the rest get skeptics by severity: two or three
for a Critical, usually none for a Low. A smaller budget cuts agents and rounds. Every lens still
runs over the same files.

Verdicts are saved as they land and each phase writes a checkpoint, so a run that dies halfway
resumes owing only what's still unverified. A crashed skeptic doesn't count as a vote, and a finding
that can't get enough real verdicts is marked unverified. The scorecard counts those and says why
the run stopped. A run that hit its budget never gets called exhaustive.

## License

[MIT](LICENSE)
