# Resume a Plan, Review It, and Ship It

Use this when an earlier agent session ran out of context in the middle of a written implementation plan, and you want a new session to finish the plan, review the whole branch, test, merge, deploy and check production, with no hand-holding.

Replace the `<...>` parts.

```
Continue <path/to/plan.md> with <your plan-execution skill, e.g. subagent-driven development>.
A previous session ran out of context.
Read the plan and the spec sections it links (<sections>) first.

State of branch <branch> (do not re-branch, do not redo done work):
- Done and committed: <Task 1 (sha)>, <Task 2 (sha)>, ...
- In progress, uncommitted: <file> has <N> new tests for <what>. The code is not written yet.
  Run them, confirm they fail for the right reason, implement the fix, get them green,
  run that task's review again, and commit.
- Then do Tasks <N..M>.

Rules:
- Run tests only on <the local test environment>. Never on <production host>.
- <Any fixture or config rule tests must follow, with an example commit.>
- Security comes first: <the access rule this feature must keep>. No new permission bypasses
  on a user request path. No test or bypass endpoints in app code.
- Do not weaken an existing test to make a change pass. If an old test relied on <old role or
  behaviour>, change only its setup, and say so in the report.
- <Any task that needs evidence, e.g. before and after screenshots as two different roles.>
- Commit after each task, so a later session can pick up if this one runs out of context.

When all tasks are done:
1. Self-review: run a final whole-branch review, and check the diff against the spec and
   <the repo's review checklist>. Fix every real finding.
2. Tests: run <the full list of test modules and frontend tests>. All must pass.
3. If any review finding or test is still open, fix it and repeat steps 1 and 2.
   Do not move on with a failure.
4. When everything passes: open the PR, merge it into <main>, and deploy with
   <deploy command> (<any PATH or tool-version setup>).
5. If the deploy fails, find the cause, fix it on a new branch, and go through steps 1 to 4 again.
6. After the deploy, check:
   - migrations succeeded on every site, all services are active, and every site answers 200;
   - <the new schema item, role or setting> exists;
   - <one read-only command that shows a known production record with its expected values>;
   - <anything you will decide yourself, e.g. "list the users and their roles (names only).
     Do not give the new role to anyone: I will choose.">

Stop and ask me, without deploying, only if a fix needs a product decision, a change to the
spec, or a weaker test.
Report: the release, the tests run, the review findings and their fixes, any change from the
plan with the reason, and the post-deploy checks.
```

## Why it works

- The exact branch state (done commits, uncommitted work) stops the new session from redoing or losing work.
- "Commit after each task" makes the next resume cheap, too.
- The loop in steps 1 to 3 has a clear exit: no open finding and no failing test.
- The stop conditions are narrow (product decision, spec change, weaker test), so the agent decides everything else and records it, instead of waiting on you.
- The post-deploy checks are read-only and name expected values, so "it deployed" is shown with evidence.
- Keep decisions about people (who gets access) with the human.
