# Off-Box Asset Build Prompt

A brief for an AI coding agent that owns a web app's frontend asset build (JS/CSS
bundles, fonts, images) end to end: time, memory, disk, output, and what the
production box pays at deploy. The goal is that the box never runs the build:
CI builds one checksummed release per commit, the box downloads, verifies and
swaps it in, and browsers receive exactly the same files as before. Written for
a multi-app framework whose build runs several bundlers (esbuild, vite,
webpack) from per-app lockfiles; adapts to any stack with a build step.

It assumes a measurement harness exists or will be written: a script that runs
the build under `/usr/bin/time -v` while sampling the whole process tree's RSS,
an inventory of every file the static location serves (bytes, gzip bytes,
sha256), and a headless browser that loads key pages as two users. The notes
at the end list the traps such a harness and such a pipeline hit.

---

## Copy-Paste Prompt

````markdown
ROLE
You own the asset build end to end: time, memory, disk, output and runtime
footprint. Cut the production box's share to zero without changing what the
browser receives.

CONTEXT
- Repository: <repo path>, branch <branch>. Read <readme>, <upstream/fork notes>,
  <perf baseline docs>, <ops runbook> and <harness dir> first.
- Perf environment: <bench/site>; existing build measurement: <script>.
  Baseline: <peak memory of the build> on <machine>.
- Production box: <size>. Asset hosts to compare: the box's web server,
  <object storage behind a CDN>, <static pages host>.
- Wait for load average under <N> before any timed run, and record it at the
  START of the run (not only at the end).

TARGET
1. The box never runs node, the bundlers or the framework's build command. It
   receives built assets as an artifact.
2. CI builds on push to <main branch>, reproducibly, one artifact per release
   with a checksum and a per-file manifest.
3. Measured gains in build time, peak memory, output size and number of files
   served, with identical bundles: same content hash, or every byte difference
   explained.

RULES
1. Fix defects where they live. Build config, package.json, bundler options and
   asset manifests in any app may be fixed in place (minimal diff, recorded with
   file, reason, measured gain). The framework's build pipeline code is
   proposal-only: a patch plus measured gain, not applied. No change to any
   page's JS/CSS behaviour; explain every changed byte.
2. Do not touch process counts, the database, the cache server, imports or
   request paths.
3. Every change: measure before and after (wall time, peak summed RSS,
   node_modules size, output size, file count, content hashes of the ten
   largest bundles). Revert when nothing improves or a bundle changes
   unexplained.
4. One change per commit; the message states before, after, how to revert.
5. Report failures as failures. No test cleanup endpoints or debug routes.
6. Risks and deferred items: dated one-liners in <roadmap file>.
7. Proof of no business change before any commit: hash-compare every bundle
   before/after; load <4-5 key pages> with the new assets as an admin and as a
   least-privileged user, cold and warm, with zero console errors and zero
   404s (attach console and network output); the asset manifest lists the same
   bundle names; repeat the browser check for every asset host you set up.
8. Never post anything upstream: no issues, PRs or pushed branches.
9. Stop and ask, with the measured number on each side, before: dropping an
   app's assets because no enabled feature uses them; choosing the asset host
   and how a release invalidates old files; a paid CI runner size. Ask before
   any upload to a cloud account; never write credentials into the repo, a
   doc or a commit.

TASK
1. Measure the build as is, per app: output size, bundle count, which pages
   reference each bundle, which apps' bundles the key pages load.
2. Find waste: duplicated libraries, dev deps installed, assets for disabled
   features, source maps shipped, node_modules reachable over HTTP,
   uncompressed text files, a bundler service process kept alive after it is
   done.
3. Fix what the rules allow, one measured change at a time.
4. CI workflow: lockfile-frozen installs, build, the rule 7 proof headless,
   upload the release (archive + sha256 + manifest) as a versioned artifact,
   publish to the CDN target behind one switch. A box script that downloads,
   verifies, unpacks and swaps the release without restarting app servers.
   Document both in the runbook and remove the build step from it.
5. Measure each asset host on the key pages: requests and bytes the box
   serves per page, page times cold and warm.
6. Write proposals for the rest.

DELIVERABLE
<doc path>: before/after table, per-app breakdown, waste found and actions,
hash comparison, rule 7 output per host, host comparison table, CI pipeline,
open decisions, handover.

DO NOT
- Change request paths, imports, process layout or database config.
- Commit node_modules, built assets or any artifact into the repo.
- Report an estimate as a result, or a planned CI run or upload as passed.
````

---

## Traps found doing this (2026-11, a Frappe/ERPNext monorepo)

- **Check the "asset URL prefix" assumption first.** Many frameworks write
  root-relative `/assets/...` URLs in server templates, in a client-side
  manifest lookup and in CSS. A CDN then either takes `/assets/*` on the same
  origin (an edge route in front of the box, no code change) or needs URL
  rewriting in code on several layers.
- **Bundle names that depend on the build folder.** A bundler plugin that puts
  absolute paths into source maps makes the hashed file names differ between
  a laptop and CI for identical code. Compare by masking the hash, then find
  the plugin.
- **Build-time configuration baked into bundles.** Some frontends read server
  config (a websocket port) at build time; a release built with a different
  value differs in one byte and renames every dependent chunk. Build with
  production's values and record them.
- **Manifest key order.** Parallel builds merge their manifests in finishing
  order: same mapping, different bytes, run to run. Compare manifests as
  mappings, or canonicalise them when packing.
- **Workers cache the asset manifest.** After swapping files, app workers may
  keep the old manifest in a per-process cache and render pages naming bundles
  that no longer exist. Flush the shared cache (or restart) as part of the
  swap, test it, and keep the previous release reachable as a fallback.
- **The static location follows symlinks into node_modules.** Tens of
  thousands of files become reachable over HTTP; pages usually load a handful
  of paths. Ship only those.
- **CI runner disk.** Installing every app's node_modules at once can exceed a
  14 GB runner; build one app at a time and drop its install and the package
  cache after its build. Emulate the runner locally in a cgroup
  (`systemd-run --user --scope -p CPUQuota=200% -p MemoryMax=7G
  -p MemorySwapMax=0`) from an empty package cache.
- **Frozen lockfiles everywhere.** An env var may not reach nested installs
  started by postinstall scripts; a config file at the build root
  (`.yarnrc: --install.frozen-lockfile true`) does. Fail the build if any
  lockfile or package.json changed.
- **Reproducible archive.** Sort entries, fixed mtime (commit time), owner 0,
  no build timestamp inside the manifest; name the release by commit plus a
  content id so two different outputs never share a name.
- **Cache rule for files without a content hash** (fonts, images, sounds, sprites): `no-cache` sends a batch of
  revalidations to the origin on every warm load for ever and keeps a CDN from caching them; a one-year max-age leaves
  changed branding stale for a year. A short max-age (one hour) gives one-year traffic inside the hour, one batch of 304s
  after it, and a release visible to everyone within the hour. Hashed bundles stay `immutable` for a year.
- **Precompress in CI, not on the box.** gzip -9 copies beside the plain files (keep the plain ones for clients without
  gzip), only text types of 1 KiB or more, served with nginx `gzip_static`, on-the-fly gzip as the fallback; measure the
  web server's CPU per cold page before and after.
- **Measure the bundler's service process.** A bundler's long-lived service can
  sit at ~1 GB beside the largest later build; stopping it once done is a
  one-line framework proposal.
- **Harness safety nets.** A measurement script that restores tracked files
  changed during its run will silently revert edits you make meanwhile; write
  only untracked files while it runs.
- **Copy-on-write filesystems.** A full install of many apps can run a btrfs
  volume out of metadata space (ENOSPC with data space free); watch
  `/sys/fs/btrfs/<uuid>/allocation/metadata/bytes_used` and stop the build
  before the limit.
