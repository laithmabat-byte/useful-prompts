# Background Job Cost Audit Prompt

A brief for an AI coding agent that owns the background job path of a
database-backed web application (a queue worker such as RQ, Celery or
Sidekiq, plus the scheduler that enqueues periodic jobs) and is accountable
for each job's SQL statement count, cache-server (Redis) command count,
memory and wall time. It measures a few jobs production runs often, groups
every statement and command by the function it comes from, separates what
the job does from what every job pays (process start, imports, connection
setup, cache refills, the queue library's bookkeeping), fixes what the
change rules allow, proves the jobs' business output did not change, and
writes the rest up as measured proposals.

Written for a framework-based app on vendored upstream packages (Frappe/
ERPNext-style, Django/Odoo-style) where the product owns one package and
other teams own process layout, config and imports. It assumes a dedicated
perf environment with its own worker, scheduler, database and Redis.

---

## Copy-Paste Prompt

````markdown
ROLE
You own the background job path end to end. You are accountable for its
SQL count, its Redis count, its memory and its wall time per job.

UNIT: worker caches

CONTEXT
- Repository and harness: <repo path>; read <baseline doc>, <earlier perf
  docs> and <harness dir> first. Branch <branch>.
- Use the perf environment only, never the dev one. If its restart re-copies
  code from the working tree, measure proposals by applying them to the
  working tree and reverting after.
- Another workload may share the host. Wait for load average under <N>
  before any timed run, and mark any run taken under load as loaded and
  unused.
- Jobs run in worker processes. Find out whether the worker forks a fresh
  process per job (RQ's default Worker, Celery prefork with
  max-tasks-per-child=1) or runs jobs in one long-lived process: it decides
  which caches can survive from one job to the next. Measure three jobs
  production runs often: <a scheduler tick with every periodic event due>,
  <an email/notification queue flush>, <a document job such as a ledger
  repost>. Add <harness>/jobs.py: it enqueues the job, waits for it, and
  reports SQL count, Redis count, wall time and worker RSS before and after.
  Run each job 20 times in one worker and report the first run and the
  steady-state median separately.
- Also measure: worker RSS after 1, 50 and 200 jobs, Redis reads per job of
  keys the previous job already read or wrote, and how many hook/config and
  schema (meta) loads repeat per job.
- Target: <X> percent fewer SQL statements or Redis commands per
  steady-state job, or a measured wall-time or RSS gain. No job slower than
  before. Stop when the rest sits in code the rules make proposal-only, and
  say so.

RULES
1. Fix defects where they live, within the change rules of the repo
   (<which packages may be edited in place, which only through hooks or as
   proposals>). Behaviour changes, and anything inside accounting or stock
   posting logic: proposal only.
2. Do not touch config, process counts, worker counts or imports; other
   tasks own those. Measure them as proposals if they are the biggest cost.
3. Every change: run jobs.py for the three jobs before and after, and the
   full page harness once at the end. Revert only when neither SQL, Redis,
   RSS nor wall time improves, or a test fails.
4. One change per commit; the message states before, after, how to revert.
5. Report failures as failures. No test cleanup endpoints or debug routes
   in application code.
6. Risks and deferred fixes go as dated one-liners in <roadmap file>.
7. Proof of no business change, before any commit: for each job capture
   what it wrote (rows inserted or updated, documents touched, mails and
   notifications queued, realtime messages), normalised, before and after;
   any difference is a failure. Never skip a permission check; jobs run as
   the user who enqueued them. A cache kept across jobs must be invalidated
   the way the framework's own caches are, and never hold user- or
   site-specific data across tenants: prove it with a test that changes the
   source record between two jobs.
8. Decisions that are the user's: stop and ask, with the measured number on
   each side.

TASK
1. For each job, group SQL and Redis by origin function. Top 10 origins,
   first run and steady state.
2. For each, say why it runs: cache rebuilt per job that could be kept per
   worker, repeated config/schema load, N+1 loop, repeated get_value, data
   the job never uses, or needed.
3. Fix what the rules allow. Measure each.
4. Write proposals for the rest, each with measured gain and business impact.

DELIVERABLE
<docs path>/<unit>.md with one row per origin and job (job, origin, SQL and
Redis before/after, why, fix, ms, RSS, severity, effort, touches upstream,
status), the RSS-by-jobs table, the proposals, the rule-7 proof (diff
commands and their empty output) and the final page counts.
````

---

## Method Notes (what made this work, and the traps)

**Keep setup out of the measured window with a hold queue.** Queueing the
emails or submitting the document that enqueues the job is not the job.
Make the framework's enqueue put the job on a private hold queue during
setup (patch the function that picks the queue), then inside the window
move it to the queue the worker reads with the queue library's own
`enqueue_job`. The job payload (function, kwargs, user, id) stays exactly
the one production builds. Count what the harness itself sends in the
window (the move, status polls) with a client-side counter and subtract it
from the server counters (MariaDB `Questions`, Redis `INFO commandstats`).

**Trace inside a fork-per-job worker.** Run a second worker process the
same way the real one starts (import the same CLI/app modules first, so
the forked child inherits the same modules), on a queue only the harness
writes to. Patch the Redis client at the socket-command layer (redis-py:
`AbstractConnection.send_command` and `pack_commands`, which also catch
connection setup and pipelines) and the DB driver's `query`, `commit`,
`rollback`, `close` (a disconnect counts as a statement in MariaDB
`Questions`). Start recording in the child's per-job entry point and dump
to a file before the child exits with `os._exit`. Record the worker
process's own commands too (dequeue, registries, heartbeats) and attribute
them to runs by timestamp, not by job id: periodic jobs reuse the same job
id every tick, so per-id files get overwritten.

**Check the trace against the server every run.** Totals must equal the
server's counters for the window; known gaps (connections the garbage
collector closes) should be the only difference. If the timing pass and
the trace disagree for the same job, look for state, not code: a cache
with a TTL (a document kept for an hour) makes "the same job" cost more for
a record nobody read recently, and a migrate or cache clear makes the next
run rebuild everything.

**Expect the fixed per-job cost to dominate.** In a fork-per-job worker a
job that does nothing can spend most of its time importing modules the
parent never imported (profile one child with cProfile), and every
process-local cache starts empty, so every Redis read of a steady run can be
a key the previous job already read. The queue library's own bookkeeping
(status hashes, registries, results, heartbeats) can be half the Redis
commands. Measure two alternatives as proposals: preloading the job modules
in the parent before it forks (copy-on-write shares them), and a worker
that runs every job in one process. Report the memory of both honestly:
preloading grows the parent but shrinks each child's private memory; a
no-fork pool can add a supervisor process and helper processes that cost
more than the whole forking worker.

**Effects capture (rule 7) for jobs.** After each run, outside the window,
read every row whose `modified` is at or after the window start in every
table (and in any side database the framework keeps logs in), the row
counts of ledger and log tables, the mails an SMTP sink in the harness
received, and the realtime messages from the trace. Normalise: names of
rows the run made become placeholders numbered by row content (schedulers
shuffle job order), run timestamps, uuids, message ids, MIME boundaries
and the framework's random row names (they can be all digits or all
letters). Mask derived state that follows the clock rather than the job:
a cron that renames ledger rows, a flag that a reversal copies from the
row it reverses. Rows a job only re-stamps (only `modified` changes) depend
on history; count them separately and do not diff them. Run two rounds on
the same code first: the files must be byte-identical. Capture effects in
every pass that runs different code or a different worker, including a pass
that runs only in the tracing worker; a variant measured without them has
no proof.

**A cache that survives between jobs needs an invalidation test.** Run
three jobs in the persistent worker; between the first and the second
create a rule the job reads through the framework's cache (a notification
on a status change), between the second and third disable it; the second
must act on it and the third must not. Check in the trace that the first
job really served that rule from process memory, so the test is not
passing trivially.

**Guard every long run against the world moving under it.** Record the
branch and a hash of the app and harness files at the start and end of a
run and mark the run invalid if they differ: another session that checks
out a branch in the same working tree mid-run makes later subprocesses run
different harness code. After a proposal run, compare the patched files
against the copy the workers actually run from. And a stack that was
stopped across a cron boundary (minute 30, the top of the hour) makes the
next tick also run every job that came due meanwhile, which shows up as
extra rows and extra imports in that tick only.

**Clean up what the harness made, out of band.** An SMTP account created
for the flush, queued mails, notification rules: delete them in the
harness's teardown, run the teardown on SIGTERM, and give the harness a
`--cleanup` mode for an interrupted run. Never as an endpoint in the app.
