# Useful Prompts

A curated collection of reusable, generic prompts for AI coding agents, developer productivity, and system optimization that anyone can copy, adapt, and benefit from.

---

## Prompt Catalog

| Prompt | Description | File |
| :--- | :--- | :--- |
| **Claude Code Telemetry, Hooks, Plugins & System Optimization** | Sets up a 2-line live telemetry status bar (context progress bar, prompt cache hit %, session cost & burn rate, code diff counter, git worktree/branch status, live RAM/Swap/CPU/GPU metrics, 5h/7d quota countdown), a real-time Prompt Quality Indicator (`UserPromptSubmit` hook), a zero-token Auto-Formatter (`PostToolUse` hook), desktop notifications, top official plugins, and `opusplan` cost/speed optimizations. | [`prompts/claude-code-telemetry-and-optimization.md`](prompts/claude-code-telemetry-and-optimization.md) |
| **Request Path Query Budget** | Agent brief for owning one request path's SQL count, cache-server count and wall time: group every statement by origin function from a query recorder's call stacks, classify each group (N+1, repeated row read, missing cache, dead query, needed), fix within layered change rules (extension points in the framework, minimal in-place defect fixes in the domain package, proposals for behaviour changes), prove ledger/stock/billing output byte-identical with a rolled-back dump-and-diff, and measure upstream proposals for real; plus traps (server counters vs recorder, cache-state-dependent Redis counts, harness re-sync undoing patches, interleaved A/B for wall time). | [`prompts/request-path-query-budget.md`](prompts/request-path-query-budget.md) |

---

## How to Use

1. Open any prompt file from the [`prompts/`](prompts/) directory.
2. Copy the prompt block and paste it directly into your AI coding agent (such as **Claude Code** or **Antigravity**).
3. The agent will execute the steps automatically while preserving your existing machine-specific configurations.
