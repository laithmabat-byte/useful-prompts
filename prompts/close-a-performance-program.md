# Close a Performance Program Prompt

A brief for an AI coding agent that closes a multi-unit performance program
on a web application: it re-runs the baseline, turns the pile of measured
proposals into one table with decisions, adopts the changes already decided
with proof that they work in production form, measures the cost of each
additional tenant (site), runs the test suites on a proper test site, closes
the release and ops gaps the units left, cleans up the results folder
(including a history rewrite it writes but does not run) and hands the owner
a closing report with the decisions still needed.

Written for a framework-based app on vendored upstream packages (Frappe/
ERPNext-style, Django/Odoo-style) where the product owns one package, core
changes are proposal-only, and a dedicated perf environment exists with its
own harness, sites, database and Redis. Other agents may be working in the
same repository at the same time.

---

## Copy-Paste Prompt

````markdown
ROLE
You close the performance program. You own the final numbers, the proposal
table, the adoption of what was decided, and the closing report. You do not
take decisions that belong to the owner: you ask, with measured numbers and
a recommendation.

WORKSPACE
Work in a git worktree on its own branch; commit there in small commits,
one concern each, and say when it is ready to merge. Other agents have
uncommitted or in-flight work in the main checkout and other worktrees:
ignore it, never touch their files.

RULES
- Never post anything upstream (no issues, PRs, pushed branches). Framework
  core changes stay proposals (a patch file with a header); in-place fixes
  to the domain package each get a row in the fork-divergence file.
- No secrets in the repository. No test endpoints or cleanup routes in
  application code; teardown lives in standalone scripts.
- Tests verify real behaviour (exact results, state rules, permission
  walls), never coverage for its own sake.
- Every number you report comes from a run you made in this task, with its
  results folder named next to it. Start runs only when the host is quiet
  (load under a stated threshold) and record the load.
- Every change states its business impact (none / what changes for users).

DELIVERABLES, in order
1. Baseline re-run: the full harness on current main, same settings as the
   original runs; one gains table, start to now (stack memory RSS and PSS,
   per-action SQL / cache commands / ms, the heavy report, PDF, build).
2. Proposal table: every patch in the proposals folder with its measured
   gain (results folder), business impact and status (applied, proposal,
   rejected). Check each still applies to main and that groups measured
   together apply together. Decide the items left open, within the rules.
3. Adopt the decided worker/process changes in production form (service
   unit in the repository) and verify them on the bench with the same unit
   file: the restart threshold triggers, the process manager restarts it, a
   job in flight across a restart finishes, the next job runs.
4. Remove the components the owner decided to drop. Measure before and
   after with the same runs; list every record that still references them
   (formats, defaults, settings) and ask what to do with them.
5. Per-tenant cost: add a second and third site, measure web, worker,
   scheduler, Redis and DB memory plus the per-user boot payload with 1, 2
   and 3 sites live; give the owner one number: MB per additional site.
6. Tests on a proper test site (setup completed the way the test bootstrap
   expects): the product's own tests and every upstream module touched by
   applied patches; pass/fail counts per module; say which feature switches
   the suites need.
7. Release and ops gaps the units left (runtime packages the box still
   installs at deploy, build scripts that write to the repository, build
   caches inside the repository, deploy steps that must happen in order:
   mark the next deploy mandatory with its steps).
8. Results cleanup: keep the runs a document references, delete the rest,
   move raw captures to a git-ignored folder, and write the exact history
   rewrite command (filter-repo on a mirror clone). Do not run it.
9. Final review: fork-divergence file complete (check it against the diff
   to each upstream), an index page of the perf docs with the gains table,
   roadmap lines updated in place with the date, and a closing report:
   gains, per-site number, what was applied, what stays a proposal, tests
   with counts, the three biggest risks, what the owner must decide.

TRAPS (each one cost a run before)
- A persistent (no-fork) worker class may inherit a "start the scheduler in
  a thread" step from its parent. Next to a standalone scheduler, whichever
  takes the scheduler lock first keeps it; the other exits with status 0. A
  process manager that stops everything when one process exits stops the
  stack; under systemd the scheduler unit restarts in a loop. Check who
  holds the lock; test that the worker leaves it free.
- A worker started on a queue backlog reads the backlog's memory, not its
  own (one run read 351 MB idle and 420 MB after 20 ticks, against 53 and
  117 MB on a drained queue). Drain the queues before measuring any worker
  layout and record how many jobs were drained.
- A no-fork worker keeps the memory of the heaviest jobs it has run until
  it restarts. A restart threshold by job count does not bound that; say so.
- Framework restart thresholds are often module constants. A process-pool
  wrapper may not let you set them and adds processes (pool, forkserver,
  resource tracker): measure the whole control group, RSS and PSS. A thin
  command around the framework's own worker class can set the threshold.
- Warm shutdown needs one signal to the worker only (KillMode=mixed) and a
  stop timeout longer than the longest job timeout; a second signal kills
  the job in flight.
- A build script that restores tracked files with `git checkout --` destroys
  other sessions' edits. Build from an export (`git stash create`, then
  `git archive`), never write to the repository, and if tracked files change
  during the run, name them and exit non-zero.
- A harness that re-syncs app code on every stack restart undoes edits made
  during a run: do not edit the app tree while a measurement is going.
- Per-site cost: park the other sites (move their folders out), restart,
  wait for the scheduler's first periodic tick, run each live site's
  periodic jobs and a desk workload, then sample. Report (N=3 - N=1) / 2.
  New empty sites measure the fixed cost of a site, not of its data.
- Feature-switch files may nest the switches (`{"features": {...}}`): print
  the switches your test configuration actually turns on before a long run.
- Tools that count processes by directory (all browsers under a shared
  folder) pick up orphans of other benches: name the orphan and its size.
- Results references come as full names, labels, short ids, time tokens and
  wildcards. Use `git ls-files -z` (names with spaces). Validate every path
  of the history rewrite against `git log --all --name-only` before writing
  the command, and run it only when no other worktree or agent is open.
- A stat that moves by tens of MB between identical runs (file-backed pages,
  a scheduler before its first tick) is noise until a recheck repeats it.
- Per-site memory: restart the database server and empty the cache server
  before each site count. They keep the table cache, buffer pages and keys of
  sites measured earlier; one run read the database 450 MB with one site
  after a three-site run, 237 MB on a fresh server.
- Test sites: install apps that seed tree or master records (item groups,
  customer groups) after the setup wizard, and check the tree roots. An
  app's install hook that runs first can make its own record the root, and
  the wizard then builds under it.
- Read "Ran N tests", not the exit code: a legacy test base class can
  preload stale upstream test records and stop before any test runs.
- Before blaming the fork for an upstream module's failure, run that module
  on the same site against an unmodified upstream tree; say which it was.
- Record what a backlog holds (job names, counts) before you measure a worker
  on it. A backlog you cannot make again is not evidence: one report blamed
  "two site creations" for 351 MB, and two new sites later left four light
  jobs. For a memory cap, queue real heavy jobs (background reports over the
  whole history) and run the same backlog with the cap off and on.
- A memory cap checked after each job bounds what stays in the worker, not a
  job's own peak; say both numbers. Read current RSS from /proc/self/statm
  (microseconds); importing psutil for it costs the worker about 4 MB.
- A cgroup lookup for a unit that is not running yet returns an empty path,
  and /sys/fs/cgroup/cgroup.procs is then the root cgroup: start sampling
  after the unit has a control group.
- Check a runbook without running it: extract every shell block and run
  `bash -n`; replace `ssh` with a stub that saves the remote script (the last
  argument, or stdin for `bash -s`) and `bash -n` each one too. An apostrophe
  in a comment inside `ssh host '...'` closes the quote and breaks the step.

REPORTING
Concise: what changed, where (file:line), how it was verified, results
folder. Questions to the owner carry the numbers and impact of each option
and a recommendation.
````

---

## Notes

- The brief assumes earlier units left measured proposals, a harness that
  writes one results folder per run, and an owner who decides. Adjust the
  deliverables to what the program actually produced.
- The per-site number is the one most owners need for pricing and box
  sizing; measure it with the production worker layout, not the dev one.
