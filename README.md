# Audit-to-A

A reusable, tool-grounded deep-audit prompt for any codebase. It grades a repo against an
operational rubric (a deterministic weighted formula, not a vibe), records findings in a
persistent ledger so progress compounds across runs, and emits a staged remediation plan
that — when executed — provably reaches an "A."

**The full prompt and usage instructions: [audit-to-A-prompt.md](audit-to-A-prompt.md)**

Works with any agentic LLM that has shell + file access (Claude Code, Cursor, SDK agents),
with multi-agent orchestration, or in plain chat with manual command execution. Supports
.NET, Node/TypeScript, Python, Go, Java/Kotlin, Rust, and more via the census cookbook in
Appendix A.
