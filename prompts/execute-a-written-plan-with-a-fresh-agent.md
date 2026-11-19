# Execute a written plan with a fresh agent

Use this brief to hand an implementation plan to a new agent session that has no memory of the design discussion. Fill in the `<...>` parts.

```text
Execute the implementation plan <plan path>. It implements the spec <spec path>. Read both in full before you change any code. Also read the project instructions file (CLAUDE.md, AGENTS.md or similar).

Method:
- Use subagent-driven development: one fresh implementer agent per task, then one fresh reviewer agent per task. Start the next task only after the reviewer passes the current one. After the last task, run one review of the whole branch.
- Work on a new branch "<branch name>" from an up-to-date "<main branch>". Commit after each task with the commit message the plan gives.
- Follow TDD for each task: write the failing test, run it and see it fail for the expected reason, write the code, run it and see it pass.

Rules:
- The plan and the spec decide. If a step conflicts with the code you find, or a step is impossible, stop that task and report the conflict. Do not invent a different design.
- Do not change a test only to make it pass. If a test fails for a reason you did not expect, find the cause first.
- Run tests only in <where tests may run>. Never on <production host>.
- Do not add test-only, cleanup or bypass endpoints to application code.
- Do not push, deploy or open a pull request unless I ask.

Report at the end, short:
- For each task: done or blocked, the commit SHA, and the test command with its result.
- Each place where you departed from the plan, and why.
- Open questions that need my decision.
```

## Why it works

- The new agent has only the plan, the spec and the repo. The brief points to them and does not restate them, so the plan stays the single source.
- "Stop and report" on a conflict stops the agent from redesigning on its own.
- One reviewer per task catches a wrong interface before later tasks build on it.
