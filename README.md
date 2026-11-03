# Useful Prompts

A curated collection of reusable, generic prompts for AI coding agents, developer productivity, and system optimization that anyone can copy, adapt, and benefit from.

---

## Prompt Catalog

| Prompt | Description | File |
| :--- | :--- | :--- |
| **Claude Code Telemetry, Hooks, Plugins & System Optimization** | Sets up a 2-line live telemetry status bar (context progress bar, prompt cache hit %, session cost & burn rate, code diff counter, git worktree/branch status, live RAM/Swap/CPU/GPU metrics, 5h/7d quota countdown), a real-time Prompt Quality Indicator (`UserPromptSubmit` hook), a zero-token Auto-Formatter (`PostToolUse` hook), desktop notifications, top official plugins, and `opusplan` cost/speed optimizations. | [`prompts/claude-code-telemetry-and-optimization.md`](prompts/claude-code-telemetry-and-optimization.md) |

---

## How to Use

1. Open any prompt file from the [`prompts/`](prompts/) directory.
2. Copy the prompt block and paste it directly into your AI coding agent (such as **Claude Code** or **Antigravity**).
3. The agent will execute the steps automatically while preserving your existing machine-specific configurations.
