# Process Layout & Server Config Memory Tuning Prompt

A brief for an AI coding agent that owns a web stack's process layout and
server config, and is accountable for its total resident memory. It works
change by change: measure, change one thing, re-measure, keep it or revert
it. Written for a Python web app (gunicorn + background workers + scheduler
+ websocket server + MySQL/MariaDB + Redis), and adapts to similar stacks.

It assumes a measurement harness already exists: scripts that start the
stack, drive a fixed workload and write per-process memory and request
times as JSON. See the notes at the end for the pitfalls such a harness has
to handle.

---

## Copy-Paste Prompt

````markdown
ROLE
You own process layout and server config. You are accountable for the
whole stack's resident memory. Measure, change, re-measure, revert what
does not win.

CONTEXT
- Repository and harness: <repo path>; read <baseline doc> and <harness dir> first.
- Measure only on the dedicated perf environment, never the dev one.
- Baseline: <total RSS idle / after N requests>, <per-process numbers>.
- Target: whole stack under <X> MB after <N> requests, with the <K> key
  pages still under <T> ms.

RULES
1. Do not edit upstream/vendored code. Your changes go in the process file
   (Procfile/supervisor/systemd units), app server config, database and
   cache config, feature flags, and the product's own extension points. If
   a win truly needs an upstream edit, write it up as a proposal with the
   measured gain, and do not apply it.
2. Every change: run the full measurement before and after. Keep both
   result folders. Revert if memory is not lower or a page gets slower
   than <T> ms.
3. One change per commit. The message states before, after, and how to revert.
4. Disable swap for the measurements so RSS is real. Say how in the doc.
5. Report failures as failures. No test cleanup endpoints or debug routes.

TASK, in this order
1. Find long-running processes that nothing in the core product needs
   (sidecars, daemons started by install/migrate hooks, keep-alive jobs).
   Stop them from starting through config or a documented feature flag,
   including whatever restarts them. Measure.
2. App server: try fewer processes with threads (e.g. 1x4, 2x2). Record
   memory and page times, alone and under concurrent load. Pick the
   smallest that keeps pages under <T> ms. Also try preload and
   max-requests recycling.
3. Cache and queue on one Redis instance: measure, and test what breaks
   (eviction policy vs queued jobs, flushes, persistence).
4. Background workers and scheduler: one worker on all queues; scheduler
   co-located. Then document a two-box layout (web + DB + cache on box A,
   workers + scheduler on box B) and the config each box needs.
5. Websocket/realtime server: measure with it off. List exactly which
   features stop working, by testing them in a real browser, not by
   reading docs.
6. Database: find the smallest buffer pool that keeps pages under <T> ms,
   also under concurrent load. Try the default, 2x and 4x, and below the
   default. Turn off performance_schema; size max_connections to the
   worker count.

DELIVERABLE
docs/<NN>-process-layout.md with one row per change:
  area | change | severity | effort hours | expected MB | actual MB |
  page time before/after | touches upstream | risk | status
plus a final table of every process's RSS at baseline and at the end,
and the recommended process file and config for a 1 GB box and a 2 GB box.

DO NOT
- Touch SQL, imports, or the asset build; other tasks own those.
- Remove a feature. Flags are fine if documented.
````

---

## Notes that made this work

- **Swap off without root**: run the harness in a transient
  `systemd-run --user --scope -p MemorySwapMax=0` scope (every child
  inherits it), and give containers `memswap_limit` = `mem_limit`. Verify
  `memory.swap.max` is 0 in each cgroup and that per-process `VmSwap` stays 0.
- **Commit the layout under test** (process file, server config keys, DB
  and cache config) in a directory the harness installs before every start,
  and record it, with any uncommitted diff, in each run's metadata.
- **Periodic jobs skew idle samples**: a scheduler can grow by tens of MB at
  its first cron tick. Wait for that tick before sampling, or results will
  depend on the clock.
- **Know your noise**: database RSS creeps up with container uptime and drops
  when the container is recreated (a config change recreates it, so compare
  fresh against fresh). File-backed pages get reclaimed under other load on a
  workstation. Repeat a run before trusting a delta under about 15 MB.
- **RSS vs PSS**: summed RSS double-counts pages a preloaded master shares
  with forked workers; PSS splits shared pages, including with unrelated
  processes on the host. Report both, and decide on the one the change
  actually affects.
- **Expected vs actual**: compute "expected MB" from the before run only (the
  RSS of what the change removes or multiplies) before you look at the after
  run.
- **Test feature loss in a browser**: two users in two browser contexts, one
  acting and one watching, recording both "the event reached the page" and
  "the screen changed without a reload". In production mode the frontend may
  reach the websocket server through the reverse proxy, so the test needs a
  proxy that routes like production (static files, websocket path, app).
- **Check that a shared cache/queue Redis loses nothing**: enqueue jobs behind
  a long-running one, then clear the cache, run the build, and fill the cache
  under the production eviction policy. See whether the pending jobs still run.
- **Never `pkill -f <pattern>`** with a pattern that appears in your own
  command line: it kills your own shell. Match on `/proc/<pid>/cmdline` argv
  instead.

## Follow-up: the database beyond its buffer pool

```markdown
<Database> sits at <N> MB with a <P> MB buffer pool. Find where the other
memory goes: temp table size, thread cache, redo log buffer, other engines'
caches, open table cache, per-connection buffers, the allocator. Measure each.
Same rules as before.
```

- **Split the server's RSS by mapping**: read `/proc/<pid>/smaps` inside the
  container as the server's own user (root without CAP_SYS_PTRACE may not).
  InnoDB maps the buffer pool and redo log buffers with `MADV_DONTDUMP` (`dd`
  in VmFlags); glibc arenas are 64 MiB-aligned; the rest is startup blocks
  and code.
- **Name the malloc'd memory once** with performance_schema memory instruments
  (`performance_schema_instrument = "memory/%=COUNTED"`) in a diagnostic run;
  it costs ~100 MB itself, so never compare it with the other runs.
- **Recreate the DB container for every run**: its RSS creeps with uptime.
- **Cache sizes cost RSS even when unused**: MyISAM key cache and Aria
  pagecache write per-block bookkeeping at startup (~23 MB at 128 MB for the
  key cache). Check which tables actually use those engines first.
- **Smaller live caches can mean more RSS**: capping the open table cache
  made the server larger (reopen churn fragments malloc arenas). Judge by RSS,
  not by the server's own accounting.
- **Measure allocator settings on top of the kept ones**: an arena limit that
  saved 10 MB alone saved 1 MB after smaller per-connection buffers.
