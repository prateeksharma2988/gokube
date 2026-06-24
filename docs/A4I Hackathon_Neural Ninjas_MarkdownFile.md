# A4I Hackathon — Neural Ninjas

**Repository:** [ThalesGroup/gokube](https://github.com/ThalesGroup/gokube)  
**Integration branch:** `prateeksharma2988/gokube:master`  
**Go version:** 1.23.0  
**Target platform:** Windows / amd64  
**Total changes:** 37 files · +3,171 / −303 lines · 0 new external dependencies

---

## Section 1 — Pull / Merge Request

**Pull / Merge Request**

> **TODO:** Add final PR link after PR is created against `ThalesGroup/gokube:master`.

---

## Section 2 — Problem & Scope

### Context

gokube is a Windows-only CLI that bootstraps a Kubernetes development environment on a laptop. It downloads and orchestrates six runtime dependencies (minikube, docker, helm, kubectl, stern, k9s) and three helm plugins (helm-spray, helm-image, helm-push), then provisions a VirtualBox- or Hyper-V-backed minikube VM with ChartMuseum, the miniapps helm repo, and the Kubernetes dashboard preconfigured.

The typical developer workflow is:

```
gokube init        # first-time setup or re-init after version change
gokube start/stop  # daily VM lifecycle
gokube save/reset  # snapshot and restore known-good state
```

Because `gokube init` is the entry point for every developer, its performance and reliability directly affect how quickly a team member reaches a working Kubernetes environment. Before this hackathon, `init` had compounding deficiencies that turned a routine operation into a slow, fragile, and confusing experience.

---

### Problem to Solve

**1. Sequential dependency downloads — 3–10 minutes per run**  
All nine dependencies downloaded one after another. Total time was the sum of all nine. On a corporate network, this meant 3–10 minutes of blocked progress per invocation — including re-runs where nothing had changed.

**2. No download cache — unconditional re-downloads**  
`UpgradeDependencies` called `DeleteExecutable()` before every `DownloadExecutable()`, making the `os.IsNotExist` guard inside permanently dead code. Every `gokube init --upgrade` re-downloaded all nine tools (~300–400 MB) regardless of version changes.

**3. Broken progress bar UX with concurrent downloads**  
`gopkg.in/cheggaaa/pb.v2` had no multi-bar pool API. With concurrent downloads, all bars shared a single terminal row, producing unreadable flickering output. There was no tool identification, no queued-download state, and elapsed timers reflected pool start time rather than per-download time.

**4. Helm plugins not integrated with progress pool**  
The three helm plugins ran sequentially after the six main tools with standalone bars and no named waiting state. A binary path bug caused the version cache to always miss, forcing full re-downloads on every run.

**5. Broken `init` / `init -u` / `init -cu` semantics**  
Plain `init` skipped the dependency check on non-first runs, leaving missing binaries unrecovered. `init -cu` deleted runtime state but left tool binaries and metadata intact — `IsCurrentVersion` returned true for all tools, making `-cu` effectively equivalent to `-c`.

**6. `gokube reset` leaving VM stopped after restore**  
When the VM was stopped before reset, the snapshot restored successfully but the command exited silently, leaving the VM in a stopped state with no message directing the developer to run `gokube start`.

**7. VirtualBox-only workflow**  
All VM lifecycle operations were hard-coded against VirtualBox (`VBoxManage`). Developers whose machines required Hyper-V (WSL2, Docker Desktop, corporate policy) had no supported path to run gokube.

---

### In Scope

| Feature | Summary |
|---|---|
| Parallel dependency downloads | 6 main tools concurrently, semaphore cap 3 |
| Parallel helm plugin downloads | 3 plugins concurrently, dedicated pool |
| Download cache with version metadata | Per-tool `.version` files in `~/.gokube/metadata/` |
| pb/v2 → pb/v3 migration | `StartPool` multi-bar rendering |
| Named waiting-state progress bars | All 9 tools visible from first second |
| Frozen elapsed time on completed bars | `done (Xs)` via named-return defer |
| Cache-hit progress display | Clean `already up to date` — no inflated timer |
| `init` semantic correction | Unconditional dependency check |
| `init -cu` semantic correction | Full binary + metadata purge before re-download |
| `gokube reset` reliability | Always starts VM after successful restore |
| Hyper-V driver support | `--driver hyperv` as VirtualBox alternative |
| Hypervisor abstraction layer | `pkg/hypervisor` interface + two implementations |
| Pre-flight validation | Elevation, Hyper-V enabled, virtual switch existence |
| Driver configuration persistence | Driver choice survives across all commands |
| Documentation | CLAUDE.md, hackathon docs |

---

### Out of Scope

| Item | Reason |
|---|---|
| Automatic Hyper-V / VirtualBox detection | Requires host state probing; deferred |
| WSL2 driver support | Pre-existing `TODO` in codebase; independent work |
| `--download-concurrency` CLI flag | One-integer constant change; not prioritised |
| Download retry on transient failures | Deferred to future work |
| Image preloading cache | Requires minikube image API integration |
| Automated test suite | No existing test infrastructure |
| JSON manifest with per-tool checksums | High per-release maintenance cost |

---

### Assumptions

- Semaphore cap of 3 concurrent downloads is appropriate for corporate proxy environments; tunable by changing a single constant.
- The ~50% first-run speedup estimate is based on architectural analysis (wall-clock ≈ max(batch1, batch2)), not measured timing.
- `--check-ip` fixed IP (`192.168.99.100`) is VirtualBox-specific; Hyper-V assigns a dynamic IP via the virtual switch.
- Swap disk support on Hyper-V is experimental, as documented in README and CHANGELOG.

---

## Section 3 — Use Cases Delivered

### Use Case 1 — Parallel Dependency Downloads

**Problem**  
Downloads ran sequentially — 3–10 minutes blocked per `gokube init` run. A single network failure required restarting all downloads from scratch, including tools that had already completed.

**Solution**  
`UpgradeDependencies` in `pkg/gokube/gokube.go` was rewritten to launch six goroutines concurrently, capped at three simultaneous HTTP connections via a buffered channel semaphore. A `WaitGroup` blocks until all six finish; a `Mutex`-guarded `firstErr` captures failures race-free. All downloads always run to completion even if one fails, maximising the number of cached results available for faster retries.

**Impact**  
Wall-clock time drops to approximately `max(batch1_time, batch2_time)` — an estimated ~50% reduction on typical corporate networks. A partial failure no longer discards completed downloads.

```
Cold first-run terminal output:
minikube v1.38.0: 45% [=========>          ]  8.2 MiB/s  9s
helm     v3.20.0: 100% [==================]  9.1 MiB/s  done (2s)
docker   29.2.1:  28% [======>             ]  5.5 MiB/s  6s
kubectl  v1.35.0: waiting to start...
stern    1.33.1:  waiting to start...
k9s      0.50.18: waiting to start...
```

---

### Use Case 2 — Download Cache with Version Validation

**Problem**  
Every `gokube init --upgrade` unconditionally deleted and re-downloaded all nine tools (~300–400 MB), even when no versions had changed.

**Solution**  
After each successful download, a metadata file is written to `~/.gokube/metadata/<toolname>.version`. Before any download, `IsCurrentVersion(binaryPath, version)` checks binary existence and version match. On a hit, the bar completes immediately with no HTTP traffic. On a miss, the binary and its metadata file are deleted before re-downloading; `WriteVersion` is called only on success.

Three helpers were added to `pkg/download/download.go`:

| Function | Purpose |
|---|---|
| `VersionFile(binaryPath)` | Returns `~/.gokube/metadata/<name>.version` |
| `IsCurrentVersion(path, version)` | Binary exists + metadata version matches |
| `WriteVersion(path, version)` | Writes version string; creates directory on first use |
| `DeleteAllMetadata()` | `os.RemoveAll(~/.gokube/metadata/)` for clean purge |

**Impact**

```
Warm re-run (all current):              Version bump (one tool changed):
minikube v1.38.0  already up to date    minikube v1.39.0: [====] done (14s)
helm     v3.20.0  already up to date    helm     v3.20.0  already up to date
docker   29.2.1   already up to date    docker   29.2.1   already up to date
...all 9 complete in < 1 second         ...only changed tool re-downloads
```

---

### Use Case 3 — Improved Progress Bar UX

**Problem**  
With concurrent downloads, pb/v2 bars interleaved onto a single terminal row — unreadable, no tool identification, no queued state. Elapsed timers reflected pool age rather than individual download time, and finished bars kept incrementing after completion.

**Solution**  
Migrated from `gopkg.in/cheggaaa/pb.v2` to `github.com/cheggaaa/pb/v3` (v3.1.7) and its `StartPool` multi-bar API. Six bars are pre-created with named `waiting to start...` templates before `StartPool` so all tools are visible from the first second. `bar.Start()` is deliberately never called on pooled bars — confirmed from pb/v3 source to spawn a competing render goroutine causing duplicate output.

The elapsed time freeze addresses a pb/v3 internals issue: `bar.render()` sets `state.time = time.Now()` unconditionally on every tick, even for finished bars. The fix uses a named-return defer in `fromUrl` that computes `time.Since(dlStart)` at actual completion, replaces the template with a static `done (Xs)` string, and calls `bar.Finish()` — preventing further ticks from inflating the counter.

**Impact**

```
Warm re-run:                        Cold download (at completion):
minikube v1.38.0  already up to date    minikube v1.38.0  done (14s)
helm     v3.20.0  already up to date    helm     v3.20.0  done (2s)
docker   29.2.1   already up to date
```

---

### Use Case 4 — Helm Plugin Progress Integration

**Problem**  
Helm plugins ran sequentially after the six main tools with standalone bars and no named waiting state. A binary path bug caused the version cache to always miss: `localFile` pointed to the plugin root, but the actual binary installs to `<plugin-root>/bin/<exe>` — causing `IsCurrentVersion` to always return false.

**Solution**  
`UpgradeHelmPlugins` was rewritten with a dedicated 3-bar `pb.StartPool` and concurrent goroutines (no semaphore needed — three plugins, all run simultaneously). Each `InstallPlugin` accepts a `*pb.ProgressBar` and follows the same cache-check → bar-freeze → `Finish` pattern as the main tools. The binary path bug was fixed by separating `pluginDir` (used for `os.RemoveAll` and download destination) from `installedBinary` (`pluginDir/bin/<exe>`, used for version checks and metadata writes).

**Impact**  
All three plugins download in parallel with named bars. Cache hits complete in under one second. The path fix means the cache actually works — eliminating unnecessary re-downloads on every run.

---

### Use Case 5 — Correct `init` / `init -u` / `init -cu` Semantics

**Problem**  
Plain `init` skipped the dependency check on non-first runs, leaving missing binaries unrecovered. `init -cu` deleted runtime state but left binaries and metadata intact, so `IsCurrentVersion` returned true for all tools — making it identical to `init -c`.

**Solution**  
Three targeted changes to `cmd/gokube/cmd/init.go`:
1. `checkMinimumRequirements()` — made unconditional (was gated on `askForUpgrade`).
2. `upgradeDependencies()` and `upgradeHelmPlugins()` calls — made unconditional.
3. Clean block — `gokube.DeleteAllExecutables()` added after working directory deletion, which calls `DeleteExecutable()` for all six tools then `download.DeleteAllMetadata()`.

| Command | Behaviour |
|---|---|
| `gokube init` | Cache-aware check always runs. Self-healing — missing binaries re-downloaded. Cache hits complete in < 1 s. |
| `gokube init -u` | Identical code path. Flag retained for backward compatibility. |
| `gokube init -cu` | Purges all binaries + `~/.gokube/metadata/`. Forces full re-download of all 9 tools. |

---

### Use Case 6 — Reset Reliability Improvement

**Problem**  
When the VM was stopped before `gokube reset`, the snapshot restored successfully but the command exited silently, leaving the VM stopped with no indication to the developer.

**Solution**  
`start()` is now called unconditionally after a successful restore. A state-aware message is printed for both pre-reset states:

```go
// Before
if running { return start() } else { return nil }

// After
if running {
    fmt.Println("VM was running before reset, restarting...")
} else {
    fmt.Println("VM was stopped before reset, starting after restore...")
}
return start()
```

| Pre-reset state | Message | Final state |
|---|---|---|
| VM running | `VM was running before reset, restarting...` | Running |
| VM stopped | `VM was stopped before reset, starting after restore...` | Running |

---

### Use Case 7 — Hyper-V Driver Support and Abstraction Layer

**Problem**  
All VM lifecycle operations were hard-coded against VirtualBox (`VBoxManage`) with no abstraction layer. Developers on machines where Hyper-V was required (WSL2, Docker Desktop, corporate policy) had no supported path to run gokube.

**Solution**  
A new `pkg/hypervisor` package introduces a `Hypervisor` interface covering all host-side VM operations. Two implementations exist: `vboxHypervisor` (delegates to `pkg/virtualbox`) and `hypervHypervisor` (drives Hyper-V via PowerShell). All VM-lifecycle commands resolve the correct implementation once via `hypervisor.New(resolveDriver())`.

New CLI flags on `gokube init`:

| Flag | Env var | Default | Notes |
|---|---|---|---|
| `--driver` | `MINIKUBE_DRIVER` | `virtualbox` | `virtualbox` or `hyperv` |
| `--hyperv-virtual-switch` | `MINIKUBE_HYPERV_VIRTUAL_SWITCH` | `""` | Omit to use Default Switch |

Pre-flight validation (`Validate()`) for Hyper-V checks in order: (1) process is running elevated, (2) Hyper-V is enabled (`Get-VM` available), (3) if a switch was supplied, it exists. Each failure returns an actionable error message directing the developer to the corrective step.

Driver and switch are persisted to `~/.gokube/config.yaml` at the end of `init`. `resolveDriver()` reads the persisted value first, then the `MINIKUBE_DRIVER` env var, then defaults to `virtualbox`. All subsequent commands inherit the choice automatically — no flags required after init.

Driver-specific adaptations included:
- VirtualBox DHCP lease reset and fixed-IP `--check-ip` enforcement are skipped for Hyper-V (dynamic IP via virtual switch).
- Swap disk: VHDX created in `~/.minikube/machines/minikube/`; Gen 1 (IDE) vs Gen 2 (SCSI) handled automatically; in-VM device node detected by diffing the disk list before and after attach (replacing the hardcoded `/dev/sdb` assumption).

**Impact**  
Developers on Hyper-V hosts can run `gokube init --driver hyperv` from an elevated shell for a full Kubernetes environment. VirtualBox users are completely unaffected — no flag required, no migration needed.

---

## Section 4 — Implementation Overview

### Architecture

```
gokube init
    └── initRun (cmd/gokube/cmd/init.go)
          ├── gokube.ReadConfig
          ├── checkMinimumRequirements          ← unconditional
          ├── hypervisor.New(driver).Validate    ← pre-flight (Hyper-V only)
          ├── [if askForClean]
          │     ├── DeleteWorkingDirectories
          │     └── gokube.DeleteAllExecutables   ← purges binaries + metadata
          ├── upgradeDependencies
          │     └── gokube.UpgradeDependencies    ← 6-bar pool, semaphore(3)
          │           └── DownloadExecutable (per tool)
          │                 ├── IsCurrentVersion → bar: already up to date
          │                 └── download.FromUrl → bar: live progress → done (Xs)
          ├── minikube.Start (driver, switch)
          ├── [swap setup via hv.AddSwapDisk + device detection]
          ├── ChartMuseum + miniapps + dashboard
          ├── upgradeHelmPlugins
          │     └── gokube.UpgradeHelmPlugins     ← 3-bar pool, no semaphore
          │           └── InstallPlugin (per plugin)
          └── gokube.WriteConfig (driver + switch persisted)
```

### Main Packages Changed

| Package | Role | Change |
|---|---|---|
| `pkg/gokube` | Orchestration | `UpgradeDependencies` and `UpgradeHelmPlugins` rewritten with goroutines, pools, semaphore; `DeleteAllExecutables` added |
| `pkg/download` | HTTP fetch + extraction | Cache helpers added; `fromUrl` named-return defer freezes bar on completion |
| `pkg/hypervisor` | VM abstraction (new) | `Hypervisor` interface + `vboxHypervisor` + `hypervHypervisor` |
| `cmd/gokube/cmd/init.go` | Init flow | Unconditional checks; clean block purge; `--driver` / `--hyperv-virtual-switch` flags |
| `cmd/gokube/cmd/reset.go` | Reset flow | Always-start after restore; Hyper-V lifecycle via `hv.*` |
| `pkg/minikube` | minikube wrapper | `Start` accepts `driver` + `hypervVirtualSwitch`; `DownloadExecutable` gains bar + cache |
| `pkg/{helm,docker,kubectl,stern,k9s}` | Tool wrappers | `DownloadExecutable` gains bar param + cache check |
| `pkg/{helmspray,helmimage,helmpush}` | Plugin wrappers | `InstallPlugin` gains bar param + cache check + binary path fix |

### Concurrency Model

```go
// UpgradeDependencies — semaphore-based, 6 tools, cap 3
sem := make(chan struct{}, 3)   // 3 concurrent HTTP connections
var wg sync.WaitGroup
var mu sync.Mutex
var firstErr error

for _, task := range tasks {
    wg.Add(1)
    go func(t func() error) {
        sem <- struct{}{}          // acquire slot — blocks if 3 already active
        defer func() { <-sem; wg.Done() }()
        if err := t(); err != nil {
            mu.Lock()
            if firstErr == nil { firstErr = err }
            mu.Unlock()
        }
    }(task)
}
wg.Wait()
_ = pool.Stop()
return firstErr
```

All goroutines complete before any error is returned, maximising metadata files written on partial failure. `golang.org/x/sync` is not in `go.mod`; the buffered channel achieves the same cap with zero new imports and a single constant to tune.

`UpgradeHelmPlugins` uses the same WaitGroup/Mutex pattern without a semaphore — only three plugins, all run simultaneously.

### Caching Approach

```
~/.gokube/
  config.yaml                  ← runtime config (driver, k8s version, container runtime)
  metadata/
    minikube.version           ← "v1.38.0"
    helm.version               ← "v3.20.0"
    docker.version             ← "29.2.1"
    kubectl.version            ← "v1.35.0"
    stern.version              ← "1.33.1"
    k9s.version                ← "0.50.18"
    helm-spray.version         ← "v4.0.13"
    helm-image.version         ← "v1.1.0"
    helm-cm-push.version       ← "0.10.4"
```

Design decisions:
- **Per-tool independent files**: no mutex needed for concurrent writes — each goroutine writes only its own file.
- **Separate `~/.gokube/metadata/` directory**: keeps binary directory clean; natural home alongside the existing `config.yaml`.
- **Version string only** (not SHA256): zero per-release maintenance cost; SHA256 would require retrieving hashes from six different upstream release pages each version bump.

### Driver Abstraction

```
         +------------------+
         |   Hypervisor     |  interface
         |  (hypervisor.go) |
         +--------+---------+
                  |
    +-------------+-------------+
    |                           |
+-----------------+   +------------------+
| vboxHypervisor  |   | hypervHypervisor |
| (virtualbox.go) |   |   (hyperv.go)    |
+-----------------+   +------------------+
        |                      |
 pkg/virtualbox             PowerShell
 (VBoxManage)            (Hyper-V module)
```

The interface is resolved once per command via `hypervisor.New(resolveDriver())`. Adding a third driver (e.g., WSL2) requires implementing the interface and adding one `case` to `New()` — no existing command files need modification.

### Progress Bar Pooling

Key constraints discovered from pb/v3 v3.1.7 source:

- **Never call `bar.Start()` on a pooled bar** — it spawns an independent render goroutine causing duplicate output.
- `pb.startTime` initialises lazily at pool-start time; `{{etime .}}` therefore reflects pool age, not per-download time.
- `bar.render()` sets `pb.state.time = time.Now()` unconditionally on every tick, even for finished bars — root cause of the elapsed-time-keeps-growing bug.

Fix: named-return defer in `fromUrl` replaces `{{etime .}}` with a frozen static string at the moment of download completion. See Section 7 for full implementation.

---

## Section 5 — Coding Assistant Usage

### Tools Used

| Tool | Role |
|---|---|
| **Claude Code** (claude.ai/code, Sonnet 4.6 — 1M context) | Primary engineering assistant throughout the project |
| **ChatGPT** | Design discussion and documentation review |

---

### How Claude Code Was Used

**Codebase investigation and call-chain tracing**  
Traced the full execution path from `gokube init` → `UpgradeDependencies` → `download.FromUrl` → `fromUrl`, confirming all downloads were sequential, that `DeleteExecutable()` made the `os.IsNotExist` guard permanently dead code, and that no shared state existed between tool packages — establishing that concurrency was safe to introduce.

**Architecture analysis before implementation**  
Three parallelism approaches (semaphore channel, `errgroup`, mutex-serialised) and three cache storage approaches (sidecar file, `~/.gokube/metadata/` directory, JSON manifest with checksums) were evaluated against gokube's existing architecture and upstream review requirements before writing any code.

**Library source verification**  
pb/v3 source files (`pb.go`, `pool.go`, `element.go`) were read directly from the Go module cache rather than documentation. This confirmed that `bar.Start()` spawns an independent goroutine, identified that `pb.startTime` initialises at pool-start time (making `{{etime .}}` reflect pool age), and pinpointed `state.time = time.Now()` on every `render()` call as the root cause of the elapsed-time-keeps-growing bug.

**Systematic multi-file refactoring**  
The `*pb.ProgressBar` parameter threading through 14 files was executed with parallel edits. Claude Code identified that `helmspray`, `helmimage`, and `helmpush` call `download.FromUrl` directly from `InstallPlugin` — a detail missed in the initial scope pass.

**Integration and conflict analysis**  
Before merging the Hyper-V branch, a diff analysis identified the 4 overlapping files and predicted all would auto-merge based on non-overlapping line ranges. The prediction was correct — zero manual conflict resolutions required.

**Bug diagnosis**  
Five non-obvious bugs diagnosed through source-level investigation:
1. Blank `0 [` rows at pool start — no template set before `StartPool`
2. Duplicate bar output — `bar.Start()` called before `StartPool`
3. Helm plugin cache always missing — wrong binary path in `InstallPlugin`
4. Inflated elapsed time on cache-hit bars — pool-start `startTime` used as origin
5. Elapsed time growing after 100% — unconditional `state.time = time.Now()` in `render()`

**Documentation generation**  
`CLAUDE.md`, `docs/hackathon-submission.md`, `docs/hackathon-progress.md`, and this document were produced using the implementation as the source of truth.

---

### Agentic Features Used

**Claude Code CLI with MCP (Model Context Protocol)**:

- **Multi-file parallel reads and edits** — independent files processed concurrently, reducing round-trips.
- **Bash tool execution** — build verification (`go build`, `go vet`), git operations, grep across the module cache.
- **Persistent memory** — `~/.claude/projects/` preserved architecture decisions and known bugs across sessions without re-reading the full codebase each time.
- **Task tracking** — `TaskCreate`/`TaskUpdate` tracked multi-step integration work across branch creation, merge, conflict resolution, and verification.
- **GitLab/GitHub MCP** — teammate's fork inspected via `git ls-remote` / `git fetch` to understand Hyper-V branch state before integration.

**Key learnings**
- Reading library source from the module cache is more reliable than documentation for understanding subtle API behaviour, particularly around goroutine lifecycle.
- Detailed pre-merge conflict analysis (predicting which files will conflict and why) significantly reduces integration time.
- Named-return patterns in Go are underused for deferred cleanup that depends on function success/failure — the `fromUrl` fix is a clean example.

**Limitations observed**
- Claude Code cannot run the binary on the target platform (Windows). All validation was build-level (`go build`, `go vet`) plus manual execution.
- The 1M context window required active memory management across long sessions to avoid losing early architectural decisions.

---

## Section 6 — Quality & Testing

### Build Verification

```sh
cd cmd/gokube
GOOS=windows GOARCH=amd64 go build ./...   # produces gokube.exe
go vet ./...                                # zero warnings
```

Both commands produced zero errors or warnings across all integration commits.

### Manual Validation

| Scenario | Validation method | Result |
|---|---|---|
| Cold first-run | `gokube init` with no `~/.gokube/metadata/` | All 9 downloads completed; metadata written; 9 named bars visible |
| Warm re-run | `gokube init` immediately after first run | All 9 bars `already up to date`; no HTTP requests; total < 1 s |
| Partial failure retry | Kill network mid-download; re-run | Only failed tools re-downloaded; completed tools served from cache |
| Version bump (one tool) | Change one `DEFAULT_*_VERSION`; re-run | Only the modified tool re-downloaded |
| `gokube init -cu` | After a successful first run | All binaries + metadata deleted; all 9 re-downloaded |
| `gokube init -u` | After a successful first run | Identical to plain init; all cache hits |
| Helm plugin cache | Second run after first install | All three plugins show `already up to date` |
| Progress bar freeze | Download reaching 100% with others still running | Completed bar frozen to `done (Xs)`; other bars continue |
| `gokube reset` (VM stopped) | Reset when VM was not running | Snapshot restored; VM started; correct message printed |
| Hyper-V `--driver hyperv` | `gokube init --driver hyperv` (elevated shell) | VM created on Hyper-V; driver persisted; subsequent commands use Hyper-V automatically |
| Hyper-V validation — not elevated | `gokube init --driver hyperv` without elevation | Clear error directing user to run as Administrator |
| VirtualBox default preserved | `gokube init` (no flags) | Identical to pre-hackathon VirtualBox behaviour |

### Regression Testing

- VirtualBox default path verified to behave identically to the pre-hackathon baseline.
- All six VM lifecycle commands (init, start/stop, pause/resume, save/reset) verified to route through the correct hypervisor implementation.
- `gokube reset --clean` flag verified safe: restore completes before `start()` is called.

### Code Quality Improvements

- Removed six unconditional `DeleteExecutable()` calls and three `DeletePlugin()` calls that preceded each download in `UpgradeDependencies` / `UpgradeHelmPlugins`.
- Replaced hardcoded `/dev/sdb` swap device with dynamic node detection (`listMinikubeDisks` / `detectNewSwapDevice` / `formatAndEnableSwap`).
- Removed stale `os/exec` import from `init.go`.
- Added `gokube.exe` to `.gitignore`; removed committed binary from git history.
- Removed the `if !keepVM { if askForUpgrade { ... } }` nesting that obscured the upgrade path.

---

## Section 7 — Technical Documentation

### Execution Flow — `gokube init`

1. Parse flags: `--driver`, `--hyperv-virtual-switch`, `--memory`, `--cpus`, `--kubernetes-version`, etc.
2. `gokube.ReadConfig` — load persisted settings from `~/.gokube/config.yaml`.
3. Version check — if `gokube-version` in config < `GOKUBE_VERSION` (1.38.0), force `askForClean = true`.
4. `checkMinimumRequirements` — validate requested tool versions against minimum floors.
5. `hypervisor.New(driver).Validate(hypervVirtualSwitch)` — pre-flight checks (Hyper-V only).
6. Set `ipCheckNeeded` — disabled for Hyper-V (dynamic IP).
7. Confirm dialog (VirtualBox only, unless `--quiet`).
8. `minikube.Delete` — remove previous VM (errors are warnings, not fatal).
9. `resetVBLease` (VirtualBox only) — clear stale host-only DHCP leases.
10. If `askForClean`: delete working directories + `gokube.DeleteAllExecutables()`.
11. `upgradeDependencies()` — parallel download of 6 tools with 6-bar pool.
12. `minikube.Start(driver, switch, ...)` — provision VM.
13. Swap disk setup (if `--swap > 0`): `hv.AddSwapDisk` → detect device node → format + enable in VM.
14. ChartMuseum install (helm upgrade) → poll `/index.yaml` until ready.
15. miniapps helm repo add + update.
16. kubernetes-dashboard NodePort patch.
17. `upgradeHelmPlugins()` — parallel download of 3 plugins with 3-bar pool.
18. `gokube.WriteConfig(version, k8sVersion, containerRuntime, driver, hypervVirtualSwitch)`.

### Parallel Download Design

**Semaphore channel pattern** (no new dependencies):

```go
sem := make(chan struct{}, 3)  // three concurrent HTTP connections

for _, task := range tasks {
    wg.Add(1)
    go func(t func() error) {
        sem <- struct{}{}          // acquire
        defer func() { <-sem; wg.Done() }()
        if err := t(); err != nil {
            mu.Lock()
            if firstErr == nil { firstErr = err }
            mu.Unlock()
        }
    }(task)
}
wg.Wait()
_ = pool.Stop()
return firstErr
```

`golang.org/x/sync` (which provides `errgroup`) is not in `go.mod`. The buffered channel achieves the same cap with zero new imports and a single constant to tune.

**All-complete-before-error**: all goroutines run to completion even if one fails. This maximises metadata files written on a partial failure. The next retry is faster because only the failed tool(s) re-download.

### Cache Mechanism

```
DownloadExecutable(url, version, bar):
  localFile = GetBinDir("gokube") + "/" + LOCAL_EXECUTABLE_NAME

  if IsCurrentVersion(localFile, version):
    bar.SetTemplateString("already up to date")
    bar.Finish()
    return nil

  os.RemoveAll(localFile)
  os.RemoveAll(VersionFile(localFile))   // metadata always purged on miss
  download.FromUrl(url, version, ..., bar)
  return WriteVersion(localFile, version)
```

`WriteVersion` is called only on success — a crash mid-download leaves no metadata file, so the next run correctly detects a cache miss and retries. For helm plugins, `pluginDir` (used for `os.RemoveAll` and download destination) and `installedBinary` (`pluginDir/bin/<exe>`, used for `IsCurrentVersion` and `WriteVersion`) are kept as separate variables — essential for correct caching.

### Progress Bar Implementation

Four bar states, all managed via template strings:

```go
// 1. Waiting — set before StartPool
`{{ yellow "minikube v1.38.0" }} waiting to start...`

// 2. Active download — set by fromUrl on HTTP response
`{{ yellow "minikube v1.38.0: " }}{{counters .}} {{bar . | green}} {{percent .}} {{speed .}} {{etime .}}`

// 3. Completed — set by deferred closure in fromUrl (success path)
`{{ green "minikube v1.38.0" }} done (14s)`

// 4. Cache hit — set by DownloadExecutable on IsCurrentVersion == true
`{{ green "minikube v1.38.0" }} already up to date`
```

Elapsed time freeze via named-return defer in `fromUrl`:

```go
func fromUrl(...) (n int64, retErr error) {
    dlStart := time.Now()
    defer func() {
        if retErr == nil {
            d := time.Since(dlStart).Round(time.Second)
            bar.SetTemplateString(`{{ green "` + name + `" }} done (` + d.String() + `)`)
        }
        bar.Finish()
    }()
    // ...
}
```

`dlStart` is set after HTTP response headers arrive, measuring actual body transfer + archive extraction time. The deferred closure replaces the live `{{etime .}}` template with a frozen static string, preventing the pool's continued `render()` ticks from incrementing the counter.

### Hyper-V Support

**Interface** (`pkg/hypervisor/hypervisor.go`):

```go
type Hypervisor interface {
    IsRunning() (bool, error)
    Pause() error
    Resume() error
    TakeSnapshot(name string) error
    DeleteSnapshot(name string) error
    RestoreSnapshot(name string) error
    ResetNetworkLeases(hostOnlyCIDR string, verbose bool) error
    ApplyVB7Workaround() error
    AddSwapDisk(swapMB int16) error
    Validate(hypervVirtualSwitch string) error
}
```

**Factory**:

```go
func New(driver string) (Hypervisor, error) {
    switch driver {
    case DriverVirtualBox:
        return &vboxHypervisor{}, nil
    case DriverHyperV:
        return &hypervHypervisor{}, nil
    default:
        return nil, ErrUnsupportedDriver
    }
}
```

**Hyper-V validation** (`hyperv.go:Validate`):
1. `isElevated()` via `windows.GetCurrentProcessToken().IsElevated()`.
2. `Get-VM -ErrorAction Stop` availability check (confirms Hyper-V is enabled).
3. `Get-VMSwitch -Name '<switch>'` if a name was supplied.

**Config persistence**:

```go
// Written at end of init
viper.Set("minikube-driver", driver)
viper.Set("hyperv-virtual-switch", hypervVirtualSwitch)

// Read by all subsequent commands
func resolveDriver() string {
    if d := viper.GetString("minikube-driver"); len(d) > 0 { return d }
    if d := os.Getenv("MINIKUBE_DRIVER"); len(d) > 0 { return d }
    return DEFAULT_MINIKUBE_DRIVER  // "virtualbox"
}
```

### `init` Command Behaviour

| Scenario | `checkMinimumRequirements` | Download phase | VM phase |
|---|---|---|---|
| First run (no config) | Always runs | All 9 downloads | VM created |
| `gokube init` (warm) | Always runs | 9 cache hits < 1 s | VM created |
| `gokube init -u` | Always runs | 9 cache hits < 1 s | VM created |
| `gokube init -cu` | Always runs | All metadata purged → 9 full downloads | VM created |
| `gokube init --keep-vm` | Always runs | 9 cache hits | VM preserved |

### `reset` Command Behaviour

```
gokube reset
  1. ReadConfig (load driver from config.yaml)
  2. hypervisor.New(resolveDriver())
  3. hv.IsRunning()  → capture pre-reset state
  4. if running: minikube.Stop()
  5. hv.RestoreSnapshot(snapshotName)
  6. [if --clean]: hv.DeleteSnapshot(snapshotName)
  7. Print state-aware message
  8. start()  ← unconditional
```

### Directory Structure

```
cmd/
  gokube/
    main.go
    cmd/
      root.go        init.go      reset.go     save.go
      start.go       pause.go     resume.go    swap.go
      version.go

pkg/
  gokube/       ← orchestration (UpgradeDependencies, UpgradeHelmPlugins, WriteConfig)
  download/     ← HTTP fetch, archive extraction, cache helpers
  hypervisor/   ← Hypervisor interface + VirtualBox shim + Hyper-V impl
  virtualbox/   ← VBoxManage wrappers, registry, DHCP leases
  minikube/     ← minikube CLI wrappers
  helm/         ← helm CLI wrappers + DownloadExecutable
  docker/       ← docker CLI wrappers + DownloadExecutable
  kubectl/      ← kubectl CLI wrappers + DownloadExecutable
  stern/        ← stern CLI wrappers + DownloadExecutable
  k9s/          ← k9s CLI wrappers + DownloadExecutable
  helmspray/    ← InstallPlugin (helm-spray)
  helmimage/    ← InstallPlugin (helm-image)
  helmpush/     ← InstallPlugin (helm-push)
  utils/        ← path helpers, archive extraction, progress bar reader

scripts/
  setup-hyperv-switch.ps1    ← PowerShell helper for Hyper-V virtual switch setup

docs/
  hackathon-submission.md
  hackathon-progress.md
  hackathon-4-quadrant.md
  hackathon-presentation.md
  A4I Hackathon_Neural Ninjas_MarkdownFile.md

CLAUDE.md       ← architecture decisions, design constraints, known issues
CHANGELOG.md
README.md
```

### Configuration

`~/.gokube/config.yaml` (managed by viper):

| Key | Written by | Read by |
|---|---|---|
| `gokube-version` | `WriteConfig` (init) | `initRun` (version comparison) |
| `kubernetes-version` | `WriteConfig` (init) | `startRun`, `resetRun` |
| `container-runtime` | `WriteConfig` (init) | `startRun` |
| `minikube-driver` | `WriteConfig` (init) | `resolveDriver()` (all commands) |
| `hyperv-virtual-switch` | `WriteConfig` (init) | `resolveDriver()` (Hyper-V commands) |

### Future Extensibility

**Adding a third driver** (e.g., WSL2):
1. Implement `Hypervisor` interface in a new `pkg/hypervisor/wsl2.go`.
2. Add `case DriverWSL2: return &wsl2Hypervisor{}, nil` to `New()`.
3. Handle driver-specific adaptations in `initRun` (IP check, DHCP skip).

**Adjusting download concurrency**: change the single constant `3` in `make(chan struct{}, 3)` in `UpgradeDependencies`.

**Adding a fourth helm plugin**: add a bar to the pool, a goroutine, and an `InstallPlugin` call. The semaphore-free design requires no additional coordination.

---

## Section 8 — Appendix

### Useful Commands

```sh
# Build (Windows cross-compile from any OS)
cd cmd/gokube
GOOS=windows GOARCH=amd64 go build -o bin/gokube-windows-amd64.exe

# Local Windows build
cd cmd/gokube
go build    # produces gokube.exe

# Vet
go vet ./...

# View integration branch history
git log --oneline integration

# Diff vs base commit
git diff 4347ce6 integration --stat

# Check current push state
git log --oneline prateek/master
```

### Hyper-V Setup Commands

```powershell
# Enable Hyper-V (requires reboot)
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All

# Create a new external switch (see scripts/setup-hyperv-switch.ps1)
New-VMSwitch -Name "gokube-switch" -NetAdapterName "Ethernet" -AllowManagementOS $true

# Init with Hyper-V (from elevated shell)
gokube init --driver hyperv
gokube init --driver hyperv --hyperv-virtual-switch "Default Switch"
```

### Important Design Constraints

- `pkg/hypervisor` **must not** import `pkg/minikube`. The driver name and switch flow into `minikube.Start` as plain parameters from the cmd layer.
- `bar.Start()` must **never** be called on a bar that will be passed to `StartPool`. It spawns a competing render goroutine causing duplicate output.
- `WriteVersion` must only be called on the **success path**. On error, no metadata file is written so the next run retries.
- `DeleteAllExecutables()` must be called **before** `upgradeDependencies()`, not after.

### Known Limitations

| Limitation | Notes |
|---|---|
| No automated test suite | gokube has no test infrastructure. All validation is manual. |
| Semaphore cap of 3 not benchmarked | Chosen conservatively for corporate proxies; tunable. |
| ~50% speedup is estimated | Based on architectural analysis, not measured timing. |
| Swap on Hyper-V is experimental | Documented in README and CHANGELOG. |
| Elapsed time on active downloads is pool-relative | `dlStart` set after HTTP headers; good approximation for first-batch tools. |
| VirtualBox and Hyper-V cannot coexist on the same host | Windows limitation; not a gokube constraint. |

### Future Improvements

| Enhancement | Notes |
|---|---|
| `--download-concurrency N` flag | Expose semaphore size as a CLI flag |
| Download retry on transient failures | Wrap `fromUrl` in a 2–3 attempt retry loop |
| `gokube version --all` from metadata | Read `~/.gokube/metadata/` without exec'ing each binary |
| WSL2 driver support | Pre-existing `TODO` in `root.go`; interface is ready |
| Automatic driver detection | Detect active Hyper-V and suggest `--driver hyperv` |
| JSON manifest with checksums | Enterprise integrity verification; adds per-release maintenance |
| Automated test suite | Significant infrastructure investment required |

### References

- pb/v3 source: `go/pkg/mod/github.com/cheggaaa/pb/v3@v3.1.7/`
- [minikube Hyper-V driver documentation](https://minikube.sigs.k8s.io/docs/drivers/hyperv/)
- [ThalesGroup/gokube](https://github.com/ThalesGroup/gokube)
- [ThalesGroup/miniapps](https://thalesgroup.github.io/miniapps)

---

*Neural Ninjas — A4I Hackathon*
