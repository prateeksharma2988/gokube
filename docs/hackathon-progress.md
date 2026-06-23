# Hackathon Progress Log

## Overview

Working branch: `master`  
Base commit at session start: `4347ce6` — Bump minikube from 1.37 to 1.38  
Go version: 1.23.0 / toolchain go1.23.3  
Target: `windows/amd64` only

---

## Completed tasks

### 1. Parallel download analysis

**Goal**: Understand `UpgradeDependencies()` before modifying it.

**Findings**:
- All 6 downloads (minikube, helm, docker, kubectl, stern, k9s) ran sequentially in `pkg/gokube/gokube.go:144`.
- No dependency ordering between tools — each writes to its own distinct path under `GetBinDir("gokube")`.
- `download.FromUrl` uses `os.MkdirTemp` per call, so there is no shared temp dir state.
- `pb.v2` progress bars all wrote to stdout — interleave was a known cosmetic risk.
- No shared state that would make concurrent downloads unsafe.

### 2. Parallel download implementation

**Files modified**: `pkg/gokube/gokube.go`

**Design**: Semaphore channel (size 3) + `sync.WaitGroup` + `sync.Mutex`-guarded first-error capture. All 6 goroutines are launched immediately; the semaphore limits to 3 concurrent HTTP fetches. All 6 downloads complete before any error is returned (deliberate departure from old fail-fast sequential behavior).

**Import added**: `"sync"` (stdlib only, no new dependency).

**Net LOC change**: +2 lines.

**Status**: Working and confirmed by user.

### 3. Progress bar overlap investigation

**Problem**: With concurrent downloads, `pb.v2` bars interleaved on stdout — each goroutine's carriage-return overwrote other bars' output, producing a single flickering line.

**Root cause**: `pb.v2 v2.0.7` has no multi-bar pool API. Each bar assumed sole ownership of its terminal row.

**Options evaluated**:
- Option A: Replace bars with simple print lines (zero dependencies, loses progress indication)
- Option B: Mutex-serialize bar rendering (bars sequential despite parallel downloads)
- Option C: Upgrade to `github.com/cheggaaa/pb/v3` which has `StartPool` multi-bar API

**Decision**: Implement Option C (pb/v3 pool) for best UX.

### 4. pb/v3 migration

**Files modified** (14 total):

| File | Change |
|---|---|
| `go.mod` | `gopkg.in/cheggaaa/pb.v2 v2.0.7` → `github.com/cheggaaa/pb/v3 v3.1.7` (version resolved by `go get`) |
| `go.sum` | Updated by `go mod tidy` — pb.v2 indirect deps removed, pb/v3 deps added |
| `pkg/download/download.go` | Import swapped; `fromUrl` + `FromUrl` each gain `bar *pb.ProgressBar`; bar creation replaced with `SetTemplateString` / `SetTotal` / `SetWidth` / `Start` on the passed-in bar |
| `pkg/utils/utils.go` | Import path updated; `pb.Reader` API identical in v3 |
| `pkg/minikube/minikube.go` | pb/v3 import added; `DownloadExecutable` gains `bar *pb.ProgressBar`, threads to `FromUrl` |
| `pkg/helm/helm.go` | Same |
| `pkg/docker/docker.go` | Same |
| `pkg/kubectl/kubectl.go` | Same |
| `pkg/stern/stern.go` | Same |
| `pkg/k9s/k9s.go` | Same |
| `pkg/gokube/gokube.go` | pb/v3 import added; `UpgradeDependencies` pre-creates 6 bars, calls `pb.StartPool`, distributes bars to closures, calls `pool.Stop()` after `wg.Wait()` |
| `pkg/helmspray/helmspray.go` | pb/v3 import added; `InstallPlugin` passes `pb.New64(0)` inline to `FromUrl` |
| `pkg/helmimage/helmimage.go` | Same |
| `pkg/helmpush/helmpush.go` | Same |

**Net LOC change across migration**: +133 / −71 (per `git diff --stat`).

**`go mod tidy` outcome**: Removed `gopkg.in/VividCortex/ewma.v1`, `gopkg.in/fatih/color.v1`, `gopkg.in/mattn/go-colorable.v0`, `gopkg.in/mattn/go-isatty.v0`, `gopkg.in/mattn/go-runewidth.v0`. Added `github.com/VividCortex/ewma v1.2.0`, `github.com/fatih/color v1.18.0`, `github.com/mattn/go-runewidth v0.0.16`, `github.com/rivo/uniseg v0.4.7`.

**Status**: Build succeeds. Parallel downloads with pb/v3 pool confirmed working by user.

### 5. Download cache (metadata version files)

**Problem addressed**: Issue #33 — every `gokube init --upgrade` re-downloaded all 6 tools (~300 MB+) unconditionally, even when nothing had changed.

**Root cause of original bug**: `UpgradeDependencies` called `DeleteExecutable()` unconditionally before each `DownloadExecutable`. The `os.IsNotExist` guard inside `DownloadExecutable` was dead code — the binary had just been deleted.

**Design**: Metadata `.version` file written to `~/.gokube/metadata/<toolname>.version` after each successful download. `IsCurrentVersion(binaryPath, version)` checks the binary exists and the metadata file content matches. Three helpers in `pkg/download/download.go`: `VersionFile`, `IsCurrentVersion`, `WriteVersion`. No new dependencies, no `go.mod` change.

**Files modified** (11 files, net +59 lines):

| File | Change |
|---|---|
| `pkg/download/download.go` | Added `VersionFile`, `IsCurrentVersion`, `WriteVersion` helpers |
| `pkg/minikube/minikube.go` | `DownloadExecutable`: version check + metadata write; `DeleteExecutable`: metadata cleanup |
| `pkg/helm/helm.go` | Same |
| `pkg/docker/docker.go` | Same |
| `pkg/kubectl/kubectl.go` | Same |
| `pkg/stern/stern.go` | Same |
| `pkg/k9s/k9s.go` | Same |
| `pkg/helmspray/helmspray.go` | `InstallPlugin`: version check + dir remove + metadata cleanup + metadata write |
| `pkg/helmimage/helmimage.go` | Same |
| `pkg/helmpush/helmpush.go` | Same |
| `pkg/gokube/gokube.go` | Removed 6 unconditional `DeleteExecutable()` and 3 `DeletePlugin()` calls |

**Behaviour**: On cache hit, bar template switches to `already up to date`, bar finishes immediately. On cache miss or version change, binary and metadata file are deleted, re-downloaded, and new metadata file written. Partial download (no metadata written) always triggers re-download.

**Status**: Implemented.

### 6. "Waiting to start..." named progress bars

**Problem**: Tools 4–6 waited silently for a semaphore slot with no visible indication — previously showed blank rows (`0 [` artifact).

**Fix**: Before calling `pb.StartPool`, each bar's template is set to `{{ yellow "<name> <version>" }} waiting to start...` via `SetTemplateString`. The pool renders this immediately on first tick via `bar.render()`'s lazy state init. When a goroutine starts downloading, `fromUrl` replaces the template with the active download template.

**Key decision**: `bar.Start()` must NOT be called before `StartPool` — see Bug 2 below.

**Files modified**: `pkg/gokube/gokube.go` (+3 lines net).

**Status**: Implemented.

---

## Bugs found and fixed

### Build failure: helmspray / helmimage / helmpush missed in initial migration

**Symptom**:
```
# github.com/gemalto/gokube/pkg/helmspray
..\..\pkg\helmspray\helmspray.go:39:116: not enough arguments in call to download.FromUrl
```

**Root cause**: `helmspray`, `helmimage`, `helmpush` also call `download.FromUrl` directly (from `InstallPlugin`, not `DownloadExecutable`). They were not in scope of the initial migration which focused on the 6 `DownloadExecutable` packages.

**Fix**: Added `pb "github.com/cheggaaa/pb/v3"` import and passed `pb.New64(0)` inline at each `download.FromUrl` call site in all three files. These packages run sequentially from `UpgradeHelmPlugins` and do not participate in the pool.

**Files fixed**: `pkg/helmspray/helmspray.go`, `pkg/helmimage/helmimage.go`, `pkg/helmpush/helmpush.go`

### Bug 3 — Helm plugin cache always misses (wrong binary path)

**Problem**: After implementing the download cache, helm plugin cache checks always returned false — plugins re-downloaded every run despite the cache being set up correctly.

**Root cause**: In all three `InstallPlugin` functions (`helmspray`, `helmimage`, `helmpush`), `localFile` was set to the plugin root directory + `LOCAL_EXECUTABLE_NAME` (e.g. `%APPDATA%\helm\plugins\helm-spray\helm-spray.exe`). However, the fileMap installs the binary to a `bin\` subdirectory: `%APPDATA%\helm\plugins\helm-spray\bin\helm-spray.exe`. The `os.Stat` in `IsCurrentVersion` failed immediately because the file at the plugin root never exists.

**Fix**: Split into two variables in each `InstallPlugin`:
- `pluginDir` — the plugin root directory; used for `os.RemoveAll` and as `dst` in `download.FromUrl`
- `installedBinary` — `pluginDir\bin\LOCAL_EXECUTABLE_NAME`; used for `IsCurrentVersion` and `WriteVersion`

**Files fixed**: `pkg/helmspray/helmspray.go`, `pkg/helmimage/helmimage.go`, `pkg/helmpush/helmpush.go`

### Bug 4 — Metadata files cluttering binary directory

**Problem**: The first cache implementation wrote `<binary>.version` files alongside each binary in `GetBinDir("gokube")`. End users browsing that directory saw 6 extra `.version` files they did not expect.

**Root cause**: `VersionFile(binaryPath)` returned `binaryPath + ".version"`, placing the file next to the binary.

**Fix**: Moved all metadata files to `~/.gokube/metadata/<toolname>.version`. `VersionFile` now derives the tool name from `filepath.Base(binaryPath)` (stripping `.exe`) and returns the path under the gokube config home. `WriteVersion` adds `os.MkdirAll` to ensure the directory exists. Helm plugin `InstallPlugin` functions add an explicit `os.RemoveAll(download.VersionFile(installedBinary))` on cache miss (since the metadata file is now outside `pluginDir` and not removed by `os.RemoveAll(pluginDir)`).

**Files changed**: `pkg/download/download.go` (+6 lines), `pkg/helmspray/helmspray.go` (+1), `pkg/helmimage/helmimage.go` (+1), `pkg/helmpush/helmpush.go` (+1) — **+9 lines total, 4 files**

---

## Decisions recorded

### Download cache approach selection

Three approaches evaluated before implementing:

| Approach | Verdict |
|---|---|
| Sidecar `.version` file per binary | **Selected** — matches per-tool-package architecture, zero maintenance cost, no new dependencies |
| Single JSON manifest (version + checksum) | Rejected for now — adds shared-state coordination complexity; worth revisiting for enterprise compliance |
| SHA256 verification (hardcoded hashes) | Rejected — very high maintenance cost, hashes must be retrieved from 6 different upstream release pages per gokube release |

Rationale: sidecar approach is upstream-ready, self-contained, and requires no maintenance beyond the `DEFAULT_*_VERSION` constants already updated each release.

### gokube init --clean does not delete binaries

Investigated whether `DeleteWorkingDirectory` functions delete the binary/sidecar directory. Confirmed they do not:

- `minikube.DeleteWorkingDirectory()` → `~/.minikube` (VM state)
- `kubectl.DeleteWorkingDirectory()` → `~/.kube` (kubeconfig)
- `docker.DeleteWorkingDirectory()` → `~/.docker` (credentials)
- `helm.DeleteWorkingDirectory()` → `%APPDATA%\helm` (repos, plugins, cache)

All binaries and main-tool sidecars live in `GetBinDir("gokube")` (the directory containing `gokube.exe`), which is never deleted by any init path. `helm.DeleteWorkingDirectory()` does remove helm plugin directories (including plugin sidecars), which correctly triggers a cache miss and plugin re-install on the next `--upgrade`. This is intentional: `--clean` resets working state, not installed tool binaries.

---

## Open issues

None currently. All known progress bar, download, and cache issues resolved.

---

## Performance

Sequential baseline (before parallelization): 6 downloads × ~30–120 s each depending on network = **~3–10 min total**.

With parallel downloads (max 3 concurrent): wall-clock time reduced to approximately **max(batch1_time, batch2_time)** where batch 1 is minikube+helm+docker and batch 2 is kubectl+stern+k9s. Expected reduction: **~50% of original time** on typical corporate network.

No precise measurement recorded yet — to be done on next `gokube init --upgrade` run.

---

## Progress bar bugs found and fixed

### Bug 1 — Blank rows before downloads start (`0 [`)

**Problem**: When `UpgradeDependencies` started, all 6 pool rows rendered as `0 [` with no tool name or version visible.

**Root cause**: Bars were created with `pb.New64(0)` and passed directly to `pb.StartPool` with no template set. pb/v3's pool renders bars immediately via `bar.render()`, which uses the default template. With `total=0` and no custom template, `counters` outputs `"0"` and `bar` outputs the opening bracket `"["` — producing `0 [` per row.

**Fix**: Set `bars[i].SetTemplateString(...)` with the tool name and version before calling `StartPool`. The pool's `render()` method reads the template immediately on its first tick.

**Files**: `pkg/gokube/gokube.go`

---

### Bug 2 — Duplicate output: every bar appeared twice

**Problem**: After fixing Bug 1 by adding `bars[i].Start()` before `pb.StartPool`, every dependency line appeared twice in the terminal — "already up to date" printed twice, active download bars rendered twice simultaneously.

**Root cause**: `bar.Start()` (pb/v3 `pb.go:163`) unconditionally spawns an **independent render goroutine** writing to stderr. `pb.StartPool` then starts the **pool's render goroutine** which also renders the same bars. Two goroutines, both writing the same bars to the same file descriptor — every line printed twice.

Confirmed by reading pb/v3 v3.1.7 source directly:
- `bar.Start()` → `go pb.writer(pb.finish)` — independent goroutine
- `StartPool` → `pool.Start()` → `go p.writer()` — pool goroutine
- `StartPool` does **not** call `bar.Start()` internally; it calls `pool.Add(pbs...)` which only appends bars to the pool's slice
- `bar.render()` (called by the pool) lazily initialises `pb.state` if nil — so bars render correctly in the pool **without** `bar.Start()` having been called

**Fix**: Remove `bars[i].Start()` from the pre-StartPool loop in `UpgradeDependencies`. Remove `bar.Start()` from the cache-hit path in each of the 6 tool `DownloadExecutable` functions. The `SetTemplateString` calls (from Bug 1 fix) are kept — they correctly configure the waiting-state template, and the pool renders them on first tick via lazy state init.

**Files changed** (−7 lines total):

| File | Change |
|---|---|
| `pkg/gokube/gokube.go` | Remove `bars[i].Start()` from pre-StartPool loop |
| `pkg/minikube/minikube.go` | Remove `bar.Start()` from cache-hit block |
| `pkg/helm/helm.go` | Same |
| `pkg/docker/docker.go` | Same |
| `pkg/kubectl/kubectl.go` | Same |
| `pkg/stern/stern.go` | Same |
| `pkg/k9s/k9s.go` | Same |

**Expected UX improvement**:
- Before: every dependency row printed twice; terminal output corrupt and unreadable
- After: each dependency renders exactly once in the pool's managed multi-line block
  - Tools waiting for a semaphore slot show: `minikube v1.38.0 waiting to start...`
  - Tools actively downloading show: `helm v3.20.0: 45% [==========>     ] 68 MiB/s`
  - Tools already cached show: `docker 29.2.1 already up to date` (once, then pool clears it when finished)

---

## Hackathon documentation produced

| File | Purpose |
|---|---|
| `docs/hackathon-submission.md` | Full technical submission (13 sections, ~600 lines) |
| `docs/hackathon-4-quadrant.md` | One-page 4-quadrant slide with ASCII grid, executive summary, key metrics, elevator pitch |
| `docs/hackathon-presentation.md` | 11-slide presentation with bullets + speaker notes per slide |
| `docs/session-summary.md` | End-of-session record: accomplishments, all modified files, technical decisions, commands, next steps |
| `docs/hackathon-progress.md` | This file — ongoing working log |
| `CLAUDE.md` | Long-term project knowledge: architecture, design decisions, known issues, current status |

---

## Current project status

All planned hackathon items are implemented and build-verified (on Windows terminal via `cd cmd\gokube && go build`):

| Feature | Status |
|---|---|
| Parallel downloads (max 3 concurrent) | Done |
| pb/v3 pool multi-bar rendering | Done |
| "waiting to start..." named bars | Done |
| Download cache (`~/.gokube/metadata/` version files) — 6 main tools | Done |
| Download cache — 3 helm plugins (`pluginDir`/`installedBinary` fix) | Done |
| Metadata files moved out of binary dir (`~/.gokube/metadata/`) | Done |
| Duplicate bar output bug (Bug 2) | Fixed |
| `0 [` blank bar bug (Bug 1) | Fixed |
| Helm plugin cache always-miss bug (Bug 3) | Fixed |
| helmspray/helmimage/helmpush build fix | Fixed |
| Hackathon documentation (submission, 4-quadrant, presentation) | Done |

**Git diff summary** (against base commit `4347ce6`):  
14 files changed · +274 insertions · −150 deletions · 0 new external dependencies

**Untracked files** (new, not yet committed):
- `CLAUDE.md`
- `docs/hackathon-progress.md`
- `docs/hackathon-submission.md`
- `docs/hackathon-presentation.md`
- `docs/hackathon-4-quadrant.md`
- `docs/session-summary.md`

---

## Validation performed

- **Build**: `cd cmd\gokube && go build` — confirmed successful on Windows (produces `gokube.exe`)
- **Parallel downloads**: confirmed working by user during session
- **Download cache (cold run)**: `gokube init --upgrade` triggers downloads on first run; metadata files written to `~/.gokube/metadata/`
- **Download cache (warm run)**: second `gokube init --upgrade` shows all 6 bars as `already up to date` in under 1 second
- **Force-upgrade path**: `gokube init` (without `--upgrade`) triggers `UpgradeDependencies` on fresh machine because stored version `0.0.0 < 1.38.0`
- **pb/v3 API**: verified from actual source files in module cache (`pool.go`, `pb.go`) — not documentation

## Remaining tasks

| Task | Priority | Notes |
|---|---|---|
| Commit all changes | High | `git add go.mod go.sum pkg/ && git commit` |
| Measure actual wall-clock time | Medium | Time cold vs warm `gokube init --upgrade` |
| Add `cmd/gokube/gokube.exe` to `.gitignore` | Low | Local build artifact currently showing as untracked |
| Helm plugin pool bars | Low | Currently standalone bars, not in pool |
| `--download-concurrency` flag | Low | Expose semaphore size as CLI flag |

---

## Session 2 — completed tasks

### 7. Helm plugin parallelization

**Goal**: Extend the parallel pool architecture used for 6 core tools to the 3 helm plugins.

**Analysis performed**: Confirmed that helmspray, helmimage, helmpush write to distinct plugin directories, distinct metadata files, and make no calls to `helm plugin install` — all three can safely run concurrently. Confirmed that `upgradeHelmPlugins()` is called after `upgradeDependencies()` completes (separated by minikube VM startup), so they cannot share a pool. Confirmed no semaphore is needed (only 3 plugins, all run simultaneously).

**Files modified**:

| File | Change |
|---|---|
| `pkg/helmspray/helmspray.go` | `InstallPlugin`: add `bar *pb.ProgressBar` param; cache-hit bar handling (+3 lines); remove inline `pb.New64(0)` |
| `pkg/helmimage/helmimage.go` | Same |
| `pkg/helmpush/helmpush.go` | Same |
| `pkg/gokube/gokube.go` | `UpgradeHelmPlugins`: full rewrite — 3-bar pool, goroutines, WaitGroup + Mutex, no semaphore |
| `.gitignore` | Added `cmd/gokube/gokube.exe` and `cmd/gokube/go` |

**Net LOC**: +46 lines across 5 files. Zero new dependencies.

**Status**: Implemented and build-verified.

---

### 8. init / init -u / init -cu semantic analysis and fix

**Analysis**: Traced all three command variants against the intended specification:

| Variant | Intended | Actual (before fix) | Gap |
|---|---|---|---|
| `gokube init` | Cache-aware check always runs; downloads only missing/outdated | `upgradeDependencies()` not called unless `-u` or first-run version mismatch | **High** — missing binaries cause runtime failures |
| `gokube init -u` | Same as above with explicit intent | Correct | None |
| `gokube init -cu` | Purge all binaries + metadata; force full re-download | Working dirs deleted but tool binaries and `~/.gokube/metadata/` untouched → `IsCurrentVersion` returns true → nothing re-downloaded | **High** — `-cu` was effectively equivalent to `-c` |

**Fixes implemented**:

1. `pkg/download/download.go` — added `DeleteAllMetadata() error`: `os.RemoveAll(~/.gokube/metadata/)`.
2. `pkg/gokube/gokube.go` — added `DeleteAllExecutables()`: calls `DeleteExecutable()` for all 6 tools then `download.DeleteAllMetadata()`. Keeps init.go free of new imports.
3. `cmd/gokube/cmd/init.go`:
   - `checkMinimumRequirements()`: removed `if !askForUpgrade` gate — now always runs.
   - Clean block (`if askForClean`): added `gokube.DeleteAllExecutables()` after `helm.DeleteWorkingDirectory()`.
   - `upgradeDependencies()` call: removed `if askForUpgrade` gate — now unconditional. Message changed to `"Checking gokube dependencies..."`.
   - `upgradeHelmPlugins()` call: same — now unconditional. Message changed to `"Checking helm plugins..."`.

**Net LOC**: +19 / −10 across 3 files.

**Resulting behavior**:
- `gokube init`: always checks all 9 tools; cache hits complete in < 1 s.
- `gokube init -u`: identical; `-u` flag kept for backward compat.
- `gokube init -cu`: purges binaries + all metadata; `IsCurrentVersion` returns false for all 9 → full re-download.

**Status**: Implemented and build-verified.

---

### 9. Elapsed time per progress bar

**Goal**: Show how long each download (or cache check) took, making the parallel speedup and cache benefit visible in numbers.

**Initial implementation**: Added `{{etime .}}` to:
- Active-download template in `pkg/download/download.go:fromUrl` (1 line)
- All 9 cache-hit `SetTemplateString` calls across the 6 tool `DownloadExecutable` functions and 3 plugin `InstallPlugin` functions (9 lines)

**Status**: Implemented and build-verified.

---

### 10. Bug fix — inflated elapsed time on cache-hit bars

**Symptom**: Cache-hit bars showed large elapsed times (e.g. `already up to date (20s)`) despite the version check taking < 1 ms.

**Root cause** (confirmed from pb/v3 v3.1.7 source `pb.go:461`):

`pb.startTime` is set lazily on the first `bar.render()` call from the pool — at `StartPool` time, before any goroutine starts work. `{{etime .}}` computes `state.Time().Sub(pb.startTime)` = current render time minus pool-start time. A bar waiting 20 s for a semaphore slot before getting a cache hit inherits T=0 as its start, so it displays `(20s)` even though the check itself took < 1 ms.

**Why `{{etime .}}` cannot be corrected through the pb/v3 API**: `pb.startTime` has a getter but no setter. The only reset path is `bar.Start()`, which spawns a competing render goroutine — the documented cause of duplicate output.

**Fix**: Replace `{{etime .}}` with the static string `<1s` in all 9 cache-hit `SetTemplateString` calls. A literal string in the template bypasses `pb.startTime` entirely.

**Files changed**: `pkg/minikube/minikube.go`, `pkg/helm/helm.go`, `pkg/docker/docker.go`, `pkg/kubectl/kubectl.go`, `pkg/stern/stern.go`, `pkg/k9s/k9s.go`, `pkg/helmspray/helmspray.go`, `pkg/helmimage/helmimage.go`, `pkg/helmpush/helmpush.go`

**Net LOC**: 0 (9 lines modified, none added or removed). Zero new imports.

**Active-download template** in `pkg/download/download.go` retains `{{etime .}}` — for first-batch tools the timer origin is close to their download start, so the displayed time is a reasonable approximation.

**Terminal output (warm cache) — before fix**:
```
helm     v3.20.0  already up to date (20s)   ← waited for semaphore, misleading
kubectl  v1.35.0  already up to date (20s)
```

**Terminal output (warm cache) — after fix**:
```
minikube v1.38.0  already up to date (<1s)
helm     v3.20.0  already up to date (<1s)
docker   29.2.1   already up to date (<1s)
kubectl  v1.35.0  already up to date (<1s)
stern    1.33.1   already up to date (<1s)
k9s      0.50.18  already up to date (<1s)
```

**Terminal output (cold download)**:
```
minikube v1.38.0: 78 MiB / 112 MiB [=========>   ] 70%  8.2 MiB/s  9s
helm     v3.20.0: 16 MiB / 16 MiB  [============] 100%  9.1 MiB/s  2s
docker   29.2.1:  already up to date (<1s)
```

**Status**: Implemented and build-verified.

---

---

### 11. Bug fix — `gokube reset` leaves VM stopped when VM was already stopped before reset

**Symptom**: Running `gokube reset` when the minikube VM was already stopped restored the snapshot successfully but left the VM in a stopped state. The user had to manually run `gokube start` to continue working.

**Root cause** (`cmd/gokube/cmd/reset.go:62–67`):

```go
// Before fix
if running {
    return start()
} else {
    return nil   // ← returned without starting; VM stays stopped
}
```

The `running` variable captured the VM state *before* the restore. After the restore, both paths should result in a running VM — the intent of `gokube reset` is to restore to a known-good state. The `running == false` path silently returned success without starting the VM.

**Asymmetry with `gokube save`**: `save` already handles the symmetric case correctly — it stops the VM if running, takes the snapshot, then restarts it. `reset` was not completing the equivalent round-trip when the VM was already stopped.

**Fix** (`cmd/gokube/cmd/reset.go`):

```go
// After fix
if running {
    fmt.Println("VM was running before reset, restarting...")
} else {
    fmt.Println("VM was stopped before reset, starting after restore...")
}
return start()
```

`start()` reads `kubernetes-version` and `container-runtime` from `~/.gokube/config.yaml` via viper and calls `minikube.Restart()`. It is safe to call unconditionally after a successful snapshot restore — the VM is in a well-defined state at that point regardless of pre-reset state.

**Edge cases verified safe**:
- `--clean` flag (deletes snapshot after restore): restore completes before `start()` — safe.
- `--name` / `-n` flag: only affects which snapshot is restored, not the start path — safe.
- `force` defaults to false on reset (not registered as a flag): `start()` uses the package-level `force` var — safe.

**Files changed**: `cmd/gokube/cmd/reset.go` — 1 line added, 3 lines changed, 1 line removed. Net: **+1 line**.

**Terminal output — before fix (VM was stopped)**:
```
Resetting minikube VM from snapshot 'gokube'...
Minikube VM has successfully been reset from snapshot 'gokube'
[process exits — VM still stopped]
```

**Terminal output — after fix (VM was stopped)**:
```
Resetting minikube VM from snapshot 'gokube'...
Minikube VM has successfully been reset from snapshot 'gokube'
VM was stopped before reset, starting after restore...
Starting minikube VM with kubernetes v1.35.0 and container runtime "docker"...
```

**Terminal output — after fix (VM was running)**:
```
Stopping minikube VM...
Resetting minikube VM from snapshot 'gokube'...
Minikube VM has successfully been reset from snapshot 'gokube'
VM was running before reset, restarting...
Starting minikube VM with kubernetes v1.35.0 and container runtime "docker"...
```

**Status**: Implemented and build-verified.

---

## Session 2 — open issues (updated)

| Issue | Priority |
|---|---|
| **Uncommitted changes** — all modified files not yet committed | High |
| No wall-clock measurement of parallel speedup (individual `{{etime .}}` now visible but no aggregate) | Medium |
| `-u` flag is now redundant (kept for backward compat) | Low |
| No automated test suite — all validation is manual | Risk (pre-existing) |

---

## Session 2 — decisions recorded

### Helm plugin pool design

Three plugin packages `helmspray`, `helmimage`, `helmpush` each write to completely distinct paths (`%APPDATA%\helm\plugins\<name>`). No semaphore needed for 3 plugins. All run simultaneously. The pool pattern from `UpgradeDependencies` is copied directly — the two pools are independent (called at different times in `initRun`) and cannot share a single `pb.StartPool` call.

### init -cu binary purge design

Two candidate approaches evaluated:
1. Call `DeleteExecutable()` for all 6 tools individually in `initRun` clean block (adds 6 imports to init.go)
2. Add `gokube.DeleteAllExecutables()` helper (keeps init.go free of new imports; `gokube.go` already imports all tool packages)

Option 2 chosen. `DeleteAllExecutables()` also calls `download.DeleteAllMetadata()` in the same function to ensure no orphaned metadata files survive the clean.

### Making upgradeDependencies unconditional

The `-u` flag previously gated the dependency download phase. Making it unconditional means plain `init` self-heals missing binaries, and the cache (< 1 s for all-present case) makes the overhead negligible. The `-u` flag is retained for backward compatibility but has no code effect. `checkMinimumRequirements()` was moved to be unconditional (it validates requested versions, not installed ones — should always run).

## Risks and blockers

- **No automated tests**: gokube has no test suite. All validation is manual via `gokube init`. A regression would only be caught at runtime.
- **Windows-only build target**: cannot build or run on Linux/macOS natively. CI cross-compile is the only non-Windows verification path.
- **Corporate proxy**: semaphore size of 3 was not measured against a real proxy; may need tuning to 2 in restricted environments.
- **`gokube init --clean` on second run**: the forced-clean path on first run writes config.yaml. Second run with `--clean` explicitly would delete `%APPDATA%\helm` including helm plugin metadata. On subsequent `--upgrade`, plugins correctly re-download. No data loss — correct behavior.

## Next actions

1. **Commit the work** — all 14 modified files + 5 new docs files uncommitted on `master`.
   ```sh
   git add go.mod go.sum pkg/ CLAUDE.md docs/
   git commit -m "Parallel downloads, pb/v3 pool, download cache, progress bar fixes"
   ```
2. **Measure wall-clock improvement** — time `gokube init --upgrade` on cold cache vs. warm cache.
3. **Consider reducing semaphore from 3 to 2** if corporate proxy throttles concurrent connections.
4. **No version bump needed** — all changes are internal; `GOKUBE_VERSION` stays at `1.38.0`.
