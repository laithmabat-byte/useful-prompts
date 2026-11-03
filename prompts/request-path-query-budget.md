# Request Path Query Budget Prompt

A brief for an AI coding agent that owns one request path of a database-backed
web application (for example "submit an invoice", "open a list", "run a
report") and is accountable for its SQL statement count, its cache-server
(Redis) command count and its wall time. It groups every statement of the
request by the function it comes from, explains each large group (N+1 loop,
the same row read repeatedly, a missing cache, a dead query, or needed), fixes
what can be fixed within the change rules, proves that business output did
not change, and writes the rest up as measured upstream proposals.

Written for a framework-based app built on upstream/vendored packages
(Frappe/ERPNext-style, Django/Odoo-style) where the product owns one package
and other teams own process layout, config and imports. It assumes a harness
that replays the request against a dedicated perf environment and captures
per-statement call stacks (an ORM query recorder or a DB slow log with stack
tags).

---

## Copy-Paste Prompt

````markdown
ROLE
You own one request path end to end. You are accountable for its SQL
count, its cache-server command count and its wall time.

UNIT: <request path, e.g. "invoice submit">

CONTEXT
- Repository and harness: <repo path>; read <baseline doc>, <earlier perf
  docs> and <harness dir> first. Branch <branch>.
- Measure only on the dedicated perf environment, never the dev one.
- Baseline per request: <page A> <n> statements, <page B> <n>, ..., <your
  unit> <n>. Cache-server counts are in <results file>. The saved query
  logs in <results dir> show every statement with its call stack.
- Target: <X> percent fewer SQL statements for your unit. No page slower
  than before.

RULES
1. Fix defects where they live, with these limits:
   - The domain/business package (upstream, vendored): a clear defect such
     as an N+1 loop, a repeated query, a dead query or a missing index may
     be fixed in place. Minimal diff, no refactors around it. Record each in
     <fork divergence doc> with file, lines, reason, business impact and
     measured gain, written so it can be sent upstream as a PR (title,
     description, test).
   - The framework package: extension points only (hooks, class overrides,
     events). Core edits go to <proposals dir> as a patch with the measured
     gain, not applied.
   - Behaviour changes, new features, or anything that changes posting /
     ledger / billing logic itself: proposal only, in either package.
2. Do not touch config, process counts or imports. Other tasks own those.
   Only your request path.
3. Every change: run the request measurement for your unit before and
   after, and the full measurement set once at the end. Revert if the count
   is not lower or any test fails. Run the product's tests and the upstream
   tests of anything you override or edit. Show that each new test fails
   without the change.
4. One change per commit. Message states before, after, business impact
   and how to revert. Every changed site carries a short comment saying what
   changed and whether it has business impact.
5. Report failures as failures. No test-cleanup endpoints or debug routes in
   application code. Correctness beats speed: a change that alters business
   output (ledger rows, totals, stock levels) is a failure however fast.
6. Anything that is a risk or a deferred fix goes as a dated one-liner in
   <roadmap file> under the right section. Not in chat only.
7. Proof of no business change, required before any commit:
   - Capture the full HTTP response of your unit before and after on 20
     different records, as two users: an administrator and a restricted
     user with a single narrow role. Diff them with timestamps and request
     ids removed. Any difference is a failure.
   - Never remove or skip a permission check. If a permission query
     repeats, cache it inside the same request only, never across users or
     requests.
   - If your change touches a save or submit path, verify that the same
     document events, notifications, webhooks and server-side scripts fire
     before and after, by counting them in a test.
   - Put the diff command and its empty output in the deliverable.

TASK
1. Read the saved query log for your unit. Group statements by origin (the
   innermost application function, below the ORM/database plumbing). List
   the top 10 origins by count.
2. For each, say why it runs that many times: N+1 loop, repeated read of the
   same row, missing cache, redundant permission check, dead query, or
   needed.
3. Fix the ones the rules allow. Measure each.
4. If the unit writes business records (ledger, stock, billing): prove the
   output is byte-identical before and after on <N> representative
   transactions (including a few reversals), by dumping and diffing the
   rows.

DELIVERABLE
<docs dir>/<nn>-<unit-slug>.md with one row per origin:
  origin function | statements before | why | fix | statements after |
  ms before/after | severity | effort hours | touches upstream | status
plus the upstream proposals with measured gain and business impact.
````

---

## Method Notes (what made this work, and the traps)

**Group by origin, not by statement text.** Walk each statement's call stack
from the innermost frame outward and skip ORM and database plumbing (query
builder, `get_value`/`get_all` wrappers, the recorder itself). The first frame
left is the origin. Keep the two or three most common call chains per origin:
they tell an N+1 loop (one chain, many rows) from a value re-read in many code
paths (many chains, the same row). Write this as a small script that also
diffs two captures per origin. Every "after" claim then comes with "nothing
else moved".

**Classic findings on a document-submit path:**
- A per-document bulk link/foreign-key check that writes a transaction cache
  but never reads it, so every ledger row of a posting re-reads the same
  masters. Often the biggest single origin.
- Lookup helpers (dimension lists, currency existence, settings) queried once
  per generated row. A transaction-scoped cache, dropped on commit/rollback
  and invalidated by the owning document's save/delete hooks, removes them
  without staleness. Check that nothing writes those rows with raw SQL.
- A query whose result is only used inside a later `if key:` branch, run
  before the branch with `key = NULL`. Pure dead work.
- The parent document reloaded in full (all child tables, `FOR UPDATE`) by a
  status update fired from a child ledger row. Usually a behaviour question
  (it fires on-change hooks), so it is a proposal.
- Per-request framework or feature-flag checks that look up metadata
  uncached on every API call. Small per request, large across all pages.

**Prove the business output did not change.** Post N fixed transactions plus
some reversals inside one transaction through the same entry points the UI
uses, dump every new and every changed ledger, stock and billing row,
normalise volatile columns (random names, timestamps, child-row hashes,
document names from a naming series), and roll back. Block commits during the
run. Run it twice on the same code first: the two files must be identical,
or the normalisation is incomplete. Then every change must reproduce the
same hash. Pick inputs the other harness scripts never touch (other items,
other customers, a date with no later entries), so runs stay comparable.

**Count with the recorder, not the server.** Server-side statement counters
include background jobs and scheduler ticks that land in the window; the
request recorder counts the request alone. Report both and say which one a
claim rests on.

**Cache-server counts depend on cache state.** A migrate or cache clear can
add repeated misses (a client cache that does not keep misses asks the server
again on every call). Compare before and after from the same state, for
example both right after a migrate, before blaming or crediting a change.

**Measure proposals for real, and check the patch is live.** If the harness's
stack restart re-syncs code from the working tree, patching the deployed copy
is silently undone. Apply the patch to the working tree, run the normal
harness, reverse it, keep the diff that ran next to the results, and confirm
the per-origin counts moved. An unchanged count usually means the patch did
not run.

**Wall time needs an interleaved A/B.** On a shared machine, single runs
move by tens of percent. Alternate before/after rounds with enough
iterations, drop the first (cold) iteration, and report median and
interquartile range. A page that runs none of the changed code and sends the
same statements is not slower because its median moved inside the IQR.

**Response diff harness (rule 7).** Script the unit's exact request sequence
as the UI sends it, for a fixed set of records chosen deterministically (for
example by a hash of the name) and covering drafts, records with comments,
and each kind of transaction. Keep status, headers and body whole. Drop only
headers that change on every response (date, cookies, content length) and
the framework's per-error random id. Send one unrecorded warm-up pass first,
so first-view side effects do not differ between runs. Run it twice on the
same code before trusting it. Add a few probe requests that must stay
refused (switched-off features, another module's records), so a change to
an access gate shows up as a diff. Store the captures gzipped without a
header timestamp, so equal content gives equal files.

**Per-request overhead often beats per-page work on small read pages.** A
form open that fires eight API calls pays eight times for every gate, flag
or metadata lookup, plus an end-of-request commit or rollback that does
nothing in a read-only transaction. Cache schema-like lookups (which module
a type belongs to) under keys the framework already invalidates. Keep
permission and per-user answers per request. The first statement of each
connection absorbs the lazy connect in recorder timings, so removing it
moves that cost to the next statement. Read wall time from the timing pass,
not the recorder.

**Tests on a throwaway site.** Run upstream suites on a scratch tenant/site
that the measured stack never serves (move it out of the served directory
while measuring) and drop it at the end. If product-level feature switches
break the upstream test bootstrap, give the scratch site its own switch
file rather than editing the shared one.
