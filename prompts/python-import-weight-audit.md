# Python Import Weight Audit Prompt

A brief for an AI coding agent that owns the import weight of a long-running
Python service: the memory every process (web worker, background worker,
scheduler) pays for the modules it imports. It maps what each live process
imports and who imports it, then defers what only rare features use, from the
product's own code, without editing upstream/vendored packages. Written for a
framework-based Python web app (gunicorn + RQ/Celery-style workers + a
scheduler), and adapts to similar stacks.

It assumes a measurement harness already exists: scripts that start the
stack, drive a fixed workload and write per-process memory and request
times as JSON. The method notes at the end cover what to add to it.

---

## Copy-Paste Prompt

````markdown
ROLE
You own Python import weight. Every process of the stack pays this cost at
start: web worker, background worker, scheduler. You are accountable for the
per-process baseline memory.

CONTEXT
- Repository and harness: <repo path>; read <baseline doc> and <harness dir> first.
- Measure only on the dedicated perf environment, never the dev one.
- Baseline: a web worker starts at <X> MB and reaches <Y> MB after <N>
  requests; the background worker and scheduler hold <Z> MB together.
- Target: web worker under <A> MB after <N> requests; worker box under <B> MB.

RULES
1. Do not edit upstream/vendored packages. Your changes go in the product's
   own package (its hooks or startup code), a sitecustomize, or environment
   variables. If a win truly needs an upstream edit, write it up as a
   proposal with the measured gain, and do not apply it.
2. Every change: run the import measurement and the per-process memory
   measurement before and after. Keep both result folders. Revert if memory
   is not lower, a key page gets slower, or a test fails.
3. One change per commit. The message states before, after, and how to revert.
4. Report failures as failures. No test-cleanup endpoints or debug routes.

TASK, in this order
1. Map it. Measure what the web worker really starts with: the app server's
   preload import, not only the framework's bare import. For each of the
   heaviest modules, find who imports it and whether the hot path uses it or
   only a rare feature. Check heavy libraries such as PDF and imaging
   libraries, spreadsheet writers, HTTP clients, SQL parsers, crypto
   bindings, HTML parsers, validation libraries, and SDK clients.
2. Hooks/plugins. Check what each installed app's hook or plugin file
   imports when it loads, and what the modules it names import when a hook
   first runs (per-request hooks, permission hooks, document events, boot
   hooks).
3. Growth. Find what the worker imports between its first and its Nth
   request. Separate the app's own model/controller modules (expected) from
   third-party libraries pulled in by a rare path, and from data (caches,
   allocator).
4. Fix what you can from the product's code: lazy-import shims, hook gates,
   environment variables a library honours. Measure each.
5. Worker box. Check whether the background worker and the scheduler each
   import the full app stack, and whether they can run lighter.

DELIVERABLE
A document with one row per finding:
  module or hook | who imports it | needed on hot path (yes/no/rare) |
  MB | fix | severity | effort hours | actual MB saved | touches upstream |
  status
plus per-process start and after-N RSS before and after, and a list of
upstream proposals with their measured gain.

DO NOT
- Touch process counts, server config, SQL, or the asset build.
- Remove a library that a feature uses. Lazy is fine, gone is not.
````

---

## Method notes (what worked)

**See inside the live processes, not just a fresh interpreter.**
- A fresh `python -c "import app"` misses what the app server preloads and
  everything requests import later. Measure the preload import as its own
  step.
- On Python 3.14+, `sys.remote_exec(pid, script)` (PEP 768) runs a script in
  a live process at its next bytecode. Use it to snapshot `sys.modules` and
  RSS, and to install an import recorder: wrap
  `importlib._bootstrap._load_unlocked`. For each module record the request
  or job being served, the innermost frames that led to the import, and RSS
  before/after (self = minus the modules it imported). Nothing is added to
  the app. A process blocked in a C call (e.g. a queue worker in BLPOP)
  answers when its next job arrives.
- Per-module RSS is exact in single-threaded processes and noisy in threaded
  ones. For clean numbers, replay the recorded imports in a fresh
  single-threaded interpreter in the worker's start state. The worker's
  growth then splits into code (the replay) and data (the rest); often most
  of it is data.

**Deferring imports from the product's code.**
- The product package can ship a top-level `sitecustomize.py`. Editable
  installs put the package's directory on `sys.path`, and `site` imports
  the first `sitecustomize` it finds in every process of that environment.
  An interpreter that already has one shadows it, so verify per deployment.
- Install one meta-path finder that answers only for an explicit list of
  module names, and give each a measured reason and a test.
- `importlib.util.LazyLoader` is not enough: any later `import x` reads
  `x.__spec__`, which triggers the load. Use a module subclass with
  `__getattr__` that raises for dunders, so the import system's probes and a
  second `import x` don't count as use, and runs the real code in the same
  module object on first real attribute access.
- Three kinds of entry:
  - module: withhold `__path__`, so a submodule import runs the package
    first;
  - package `__init__` that only re-exports: submodules import without it;
  - hook gate: a module named by a per-request hook gets stand-in hook
    functions that load it only for the request paths it acts on.
- Only plain `import x` bindings can be deferred. `from x import y` and
  `import x.sub` run the package immediately; those need an upstream edit.
- Check every deferral in the live process after the full workload: a
  deferral that the workload undoes saves nothing. A trace switch (an env
  var that prints the stack when a deferred module runs) finds the trigger.
- Add a kill switch (env var), and make the shim itself cost nothing at
  interpreter start: lazy-import `threading` and `traceback`.
- Tests:
  - the mechanism, on throwaway packages;
  - per entry, an AST check of the upstream code it relies on (the hook's
    path checks, the plain import, the re-export-only `__init__`), so an
    upstream change fails the test;
  - the effect in a fresh interpreter, which must fail with the shim off.

**Measuring upstream proposals without applying them.**
- Keep each proposal as exact old/new string edits.
- Compile the edited source in the parent, marshal the code, and have a
  fresh child interpreter load it under the real module name through a
  meta-path finder. Marshalled code costs what a `.pyc` load does;
  compiling source in the child leaves allocator high-water and biases
  against large files.
- Measure scenarios: worker start, worker code after the workload (replay
  only the imports request code made from function frames, so edited module
  bodies drop their edges), the scheduler's tick, the background worker.
- Write each proposal as a patch file.
- Peeling one import edge often gains nothing, because another path loads
  the same chain. Re-trace after each edit until the heavy modules
  disappear, then measure.

**Noise and the allocator.**
- A threaded worker's RSS after N requests can vary by ±8 MB between
  identical runs, from per-thread malloc arenas. Measure deterministic
  points (preload import, idle, early requests, replays), and compare the
  total with interleaved A/B runs: alternate two process environments and
  compare medians.
- `MALLOC_ARENA_MAX=2` (glibc) cut a 4-thread gunicorn worker by 16 MB and
  its spread from ~14 MB to under 1 MB, with no latency change. Check the
  p95 of rare actions against their sample count: with 10 samples, p95 is
  the max.

**Worker box.**
- CLI entry points (`<framework> worker`, `<framework> schedule`) often
  import every installed app's commands. Measure a direct entry point.
- A scheduler's first tick can pull half the framework through query
  helpers that import the UI boot module, or through sandbox modules
  imported on every query. Trace the tick in the live process.
- Know the floor: the framework's bare import and init, times the number of
  processes.
