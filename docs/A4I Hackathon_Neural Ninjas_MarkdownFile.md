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

gokube is a Windows-only CLI tool that bootstraps a Kubernetes development environment on a laptop. It downloads and orchestrates six runtime dependencies (minikube, docker, helm, kubectl, stern, k9s) and three helm plugins (helm-spray, helm-image, helm-push), then provisions a VirtualBox- or Hyper-V-backed minikube VM with ChartMuseum, the miniapps helm repository, and the Kubernetes dashboard preconfigured.

The typical developer workflow is:

```
gokube init        # first-time setup or re-init after version change
gokube start/stop  # daily VM lifecycle
gokube save/reset  # snapshot and restore known-good state
```

Because `gokube init` is the entry point for every developer on the team, its performance and reliability directly affect how long it takes to get from zero to a working Kubernetes environment.

---

### Problem to Solve

Before this hackathon, `gokube init` had several compounding deficiencies:

**1. Sequential dependency downloads (3–10 minutes per run)**
All nine dependencies downloaded one after another. Total time was the sum of all nine, not the maximum. On a corporate network with proxy throttling, this meant 3–10 minutes of blocked progress on every invocation — including re-runs where nothing had changed.

**2. No download cache — unconditional re-downloads**
`UpgradeDependencies` called `DeleteExecutable()` before every `DownloadExecutable()`. The `os.IsNotExist` guard inside `DownloadExecutable` was permanently dead code. Every `gokube init --upgrade` re-downloaded all six tool binaries (~300–400 MB) regardless of whether any version had changed.

**3. Broken progress bar UX with concurrent downloads**
The original library (`gopkg.in/cheggaaa/pb.v2`) had no multi-bar pool API. With concurrent downloads, all bars shared a single terminal row, interleaving over each other and producing an unreadable flickering line. There was no tool identification, no waiting state for queued downloads, and elapsed timers were inflated by pool-start time rather than per-download time.

**4. Helm plugins not integrated with progress pool**
The three helm plugins (helm-spray, helm-image, helm-push) ran sequentially after the six main tools, each with a standalone progress bar and no named waiting state. A critical bug caused the version cache check to always miss, forcing a full re-download of all three plugins on every run.

**5. Confusing `init` / `init -u` / `init -cu` semantics**
- `gokube init` (without `--upgrade`) skipped the dependency check entirely on non-first runs, meaning missing binaries were not recovered automatically.
- `gokube init -cu` (clean + upgrade) deleted runtime state directories but left tool binaries and all version metadata intact. `IsCurrentVersion` returned true for all tools, so nothing re-downloaded — making `-cu` effectively equivalent to `-c`.

**6. `gokube reset` leaving VM stopped after restore**
If the VM was stopped before `gokube reset`, the snapshot restored successfully but the command exited silently, leaving the VM in a stopped state. The user had to manually run `gokube start` — with no message explaining this was necessary. The recovery command did not produce a recoverable state.

**7. VirtualBox-only workflow**
All VM lifecycle operations (snapshots, pause/resume, swap disk attachment, DHCP lease management) were hard-coded against VirtualBox tooling (`VBoxManage`). There was no abstraction layer. Developers on machines where Hyper-V was already required (WSL2, Docker Desktop, corporate policy) had no supported path to run gokube.

---

### Why It Matters

The combined effect of these problems was significant developer friction:

- A developer re-running `gokube init` after a VM reset waited 3–10 minutes and received no useful feedback during that time.
- A partial failure (network drop, VirtualBox error) discarded all completed downloads and required a full restart from scratch.
- Developers who needed Hyper-V could not use gokube at all.
- The recovery command (`gokube reset`) did not reliably deliver a ready-to-use environment.

These are not edge cases — they affect every developer using gokube on every re-initialization.

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
| `init` semantic correction | Always-unconditional dependency check |
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
| Automatic Hyper-V / VirtualBox detection | Requires probing host state; deferred to future work |
| WSL2 driver support | Pre-existing `TODO` in codebase; independent of this work |
| `--download-concurrency` CLI flag | Semaphore size is a one-integer change; not prioritised |
| Download retry on transient failures | Wrapping `fromUrl`; deferred to future work |
| Image preloading cache | Requires minikube image API integration |
| Automated test suite | gokube has no test infrastructure; would require significant investment |
| JSON manifest with per-tool checksums | High maintenance cost (per-tool SHA256 per release) |

---

### Assumptions

- The semaphore cap of 3 concurrent downloads is appropriate for corporate proxy environments. It was not benchmarked against real proxy limits and can be adjusted by changing one constant.
- The ~50% first-run speedup estimate is based on architectural analysis (parallel execution → wall-clock ≈ max(batch1, batch2)) rather than measured timing.
- The `--check-ip` constraint (fixed IP `192.168.99.100`) is VirtualBox-specific and does not apply to Hyper-V, which assigns a dynamic IP via the virtual switch.
- Swap disk support on Hyper-V is experimental, as documented in README and CHANGELOG.

---

## Section 3 — Use Cases Delivered

---

### Use Case 1 — Parallel Dependency Downloads

**Description**  
A developer running `gokube init` on a machine with no tools installed, or with version-outdated tools, should not wait for each download to finish before the next begins. All downloads that can proceed concurrently should do so, subject to a connection cap that respects corporate proxy limits.

**Expected Outcome**  
Total download time for six tools drops from the sum of six sequential downloads to approximately the maximum of two batches of three — a ~50% reduction on typical corporate networks.

**What Was Implemented**

`UpgradeDependencies` in `pkg/gokube/gokube.go` was rewritten. Six goroutines launch immediately. A buffered channel `sem := make(chan struct{}, 3)` acts as a semaphore — each goroutine must acquire a slot before starting its HTTP fetch, limiting concurrent connections to three. A `sync.WaitGroup` blocks until all six complete. A `sync.Mutex`-guarded `firstErr` variable captures the first non-nil error without race conditions. All six downloads always run to completion even if one fails, maximising the number of metadata files written for a faster retry.

```
Terminal output (cold first-run):
minikube v1.38.0: 45% [=========>          ]  8.2 MiB/s  9s
helm     v3.20.0: 100% [==================]  9.1 MiB/s  done (2s)
docker   29.2.1:  28% [======>             ]  5.5 MiB/s  6s
kubectl  v1.35.0: waiting to start...
stern    1.33.1:  waiting to start...
k9s      0.50.18: waiting to start...
```

---

### Use Case 2 — Download Cache with Version Validation

**Description**  
A developer re-running `gokube init` after a version that was already installed should not transfer any data. The tool should detect that each binary is current and skip the download entirely.

**Expected Outcome**  
- Warm re-run: all 9 tools confirmed in under one second, 0 bytes transferred.
- Partial failure retry: only the tools that failed re-download; previously completed tools are served from cache.
- Version bump: only the changed tool(s) re-download.

**What Was Implemented**

After each successful download, a metadata file is written to `~/.gokube/metadata/<toolname>.version` containing the version string. Before any download, `download.IsCurrentVersion(binaryPath, version)` checks that the binary exists on disk and the metadata file content matches the requested version. On a hit, the bar is marked complete and the function returns immediately. On a miss, the binary and its metadata file are deleted, the download runs, and a new metadata file is written on success.

Three helpers were added to `pkg/download/download.go`:

| Function | Purpose |
|---|---|
| `VersionFile(binaryPath)` | Derives `~/.gokube/metadata/<name>.version` from binary path |
| `IsCurrentVersion(path, version)` | Binary exists + metadata matches |
| `WriteVersion(path, version)` | Writes version string; creates directory on first use |
| `DeleteAllMetadata()` | `os.RemoveAll(~/.gokube/metadata/)` for clean purge |

A partial download (crash before `WriteVersion`) leaves no metadata file, so the next run detects a cache miss and re-downloads correctly. The binary directory stays clean — no version files alongside executables.

---

### Use Case 3 — Improved Progress Bar UX

**Description**  
A developer running a parallel download should see one stable terminal row per tool, with the tool name, version, download progress, speed, and a meaningful elapsed time. Queued tools should be visible from the first second, not blank. Completed tools should freeze at their final state and not continue updating their timer.

**Expected Outcome**

```
Warm re-run:
minikube v1.38.0  already up to date
helm     v3.20.0  already up to date
docker   29.2.1   already up to date

Cold download (tool at completion):
minikube v1.38.0  done (14s)
helm     v3.20.0  done (2s)
```

**What Was Implemented**

The `gopkg.in/cheggaaa/pb.v2` library was replaced with `github.com/cheggaaa/pb/v3` (v3.1.7), which provides a `StartPool` multi-bar rendering API. Six bars are pre-created before the pool starts, each with a named `waiting to start...` template so all tools are visible immediately. `bar.Start()` is deliberately never called before `StartPool` — doing so spawns a competing render goroutine that causes duplicate output (confirmed from pb/v3 source).

The elapsed time freeze fix addresses a pb/v3 internals issue: `bar.render()` sets `pb.state.time = time.Now()` unconditionally on every tick, even for finished bars. `{{etime .}}` therefore keeps incrementing until `pool.Stop()` fires after the slowest goroutine. The fix uses named returns in `fromUrl` (`(n int64, retErr error)`): a deferred closure checks `retErr` at return time, computes `time.Since(dlStart)`, and replaces the template with a static `done (Xs)` string before calling `bar.Finish()`. The pool continues rendering the frozen string.

Cache-hit bars set a static template (`already up to date`) with no elapsed time — a cache check is instantaneous and the duration is not meaningful.

---

### Use Case 4 — Helm Plugin Progress Integration

**Description**  
The three helm plugins (helm-spray, helm-image, helm-push) should benefit from the same parallel execution, download caching, and named progress bar UX as the six main tools.

**Expected Outcome**  
All three plugins download concurrently with their own named progress pool. Cache hits complete in under one second and display `already up to date`. Completed downloads freeze at `done (Xs)`.

**What Was Implemented**

`UpgradeHelmPlugins` in `pkg/gokube/gokube.go` was rewritten with a dedicated 3-bar `pb.StartPool`. All three `InstallPlugin` calls run as concurrent goroutines with a `WaitGroup` and `Mutex` (no semaphore needed — only three plugins, all run simultaneously). Each `InstallPlugin` function accepts a `*pb.ProgressBar` parameter and follows the same cache-check → bar-freeze → `Finish` pattern as the main tools.

A binary path bug was also fixed: all three `InstallPlugin` functions used a `localFile` path pointing to the plugin root directory, but the actual binary is installed to `<plugin-root>/bin/<exe>`. `IsCurrentVersion` always returned false (the file never existed at the checked path), forcing full re-downloads. The fix separates `pluginDir` (used for `os.RemoveAll` and download destination) from `installedBinary` (`pluginDir/bin/<exe>`, used for version checks and metadata).

---

### Use Case 5 — Correct `init` / `init -u` / `init -cu` Semantics

**Description**  
The three variants of `gokube init` should behave predictably and as documented. Plain `init` should be self-healing. `init -cu` should produce a clean-slate result equivalent to a fresh machine.

**Expected Outcome**

| Command | Behaviour |
|---|---|
| `gokube init` | Always runs cache-aware check. Cache hits complete in < 1 s. Downloads only missing or outdated tools. Self-healing. |
| `gokube init -u` | Identical code path. Flag retained for backward compatibility. |
| `gokube init -cu` | Purges all binaries + `~/.gokube/metadata/` before downloading. Forces full re-download of all 9 tools. |

**What Was Implemented**

Three targeted changes to `cmd/gokube/cmd/init.go`:

1. `checkMinimumRequirements()` made unconditional (was gated on `askForUpgrade`).
2. `upgradeDependencies()` and `upgradeHelmPlugins()` calls made unconditional (were gated on `askForUpgrade`).
3. In the `askForClean` block: `gokube.DeleteAllExecutables()` added after working directory deletion. This calls `DeleteExecutable()` for all six main tools and then `download.DeleteAllMetadata()`, ensuring that `IsCurrentVersion` returns false for all nine tools on the next check.

`DeleteAllExecutables()` was introduced in `pkg/gokube/gokube.go` to avoid adding six tool-package imports to `init.go` — the orchestration package already imports all tool packages.

---

### Use Case 6 — Reset Reliability Improvement

**Description**  
`gokube reset` should always leave the environment in a running, usable state after a successful snapshot restore. A developer using reset to recover from a bad state should not need to know or remember whether the VM was running when they triggered the command.

**Expected Outcome**

| Pre-reset state | Output | Final state |
|---|---|---|
| VM was running | `VM was running before reset, restarting...` | Running |
| VM was stopped | `VM was stopped before reset, starting after restore...` | Running |

**What Was Implemented**

In `cmd/gokube/cmd/reset.go`, the tail of `resetRun` was:

```go
// Before
if running {
    return start()
} else {
    return nil  // VM stayed stopped with no message
}
```

Changed to:

```go
// After
if running {
    fmt.Println("VM was running before reset, restarting...")
} else {
    fmt.Println("VM was stopped before reset, starting after restore...")
}
return start()
```

`start()` reads `kubernetes-version` and `container-runtime` from the persisted config and calls `minikube.Restart()`. It is safe to call unconditionally after a snapshot restore — the VM is in a well-defined state regardless of pre-reset history.

---

### Use Case 7 — Hyper-V Driver Support and Abstraction Layer

**Description**  
Developers on Windows machines where Hyper-V is active (required by WSL2, Docker Desktop, or corporate policy) should be able to use gokube without needing to disable Hyper-V or install VirtualBox. VirtualBox users should be completely unaffected.

**Expected Outcome**  
- `gokube init --driver hyperv` provisions a minikube VM on Hyper-V.
- All VM lifecycle commands (pause, resume, save, reset, start) continue to work with Hyper-V.
- `gokube init` (no flags) continues to use VirtualBox — no migration required.
- Driver choice is persisted and automatically reused across all commands.
- Misconfiguration (not elevated, Hyper-V disabled, bad switch name) is caught before any destructive operation with a clear error message.

**What Was Implemented**

A new `pkg/hypervisor` package defines a `Hypervisor` interface covering all host-side VM operations:

```
IsRunning, Pause, Resume, TakeSnapshot, DeleteSnapshot, RestoreSnapshot,
ResetNetworkLeases, ApplyVB7Workaround, AddSwapDisk, Validate
```

Two implementations:
- `vboxHypervisor` — delegates to the existing `pkg/virtualbox` package. Error sentinels are translated to the driver-neutral `hypervisor.ErrSnapshotNotExist`.
- `hypervHypervisor` — drives Hyper-V via PowerShell (`Get-VM`, `Checkpoint-VM`, `Restore-VMSnapshot`, `Add-VMHardDiskDrive`, etc.).

New CLI flags on `gokube init`:

| Flag | Env var | Default | Notes |
|---|---|---|---|
| `--driver` | `MINIKUBE_DRIVER` | `virtualbox` | `virtualbox` or `hyperv` |
| `--hyperv-virtual-switch` | `MINIKUBE_HYPERV_VIRTUAL_SWITCH` | `""` | Optional; omit to use Default Switch |

Pre-flight validation (`Validate()`) for Hyper-V checks in order:
1. Process is running elevated (administrator privileges).
2. `Get-VM` is available — confirms Hyper-V is enabled.
3. If a switch name was supplied, the named switch exists.

Each check produces an actionable error message directing the user to the corrective step.

Driver persistence: chosen driver and switch name are written to `~/.gokube/config.yaml` at the end of `init`. `resolveDriver()` reads the persisted value first, then the `MINIKUBE_DRIVER` env var, then defaults to `virtualbox`. All subsequent commands call `hypervisor.New(resolveDriver())` automatically.

Driver-specific adaptations:
- VirtualBox host-only DHCP lease reset skipped for Hyper-V (no host-only network).
- Static `--check-ip` enforcement disabled for Hyper-V (dynamic IP).
- Swap disk: VHDX created in `~/.minikube/machines/minikube/`; Gen 1 (IDE) vs Gen 2 (SCSI) handled automatically.
- In-VM swap device node detected dynamically by diffing disk list before and after attach, rather than assumed to be `/dev/sdb`.

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
| `pkg/download` | HTTP fetch + extraction | Cache helpers added; `fromUrl` uses named-return defer to freeze progress bar on completion |
| `pkg/hypervisor` | VM abstraction (new) | `Hypervisor` interface + `vboxHypervisor` + `hypervHypervisor` |
| `cmd/gokube/cmd/init.go` | Init flow | Unconditional checks; clean block purge; `--driver`/`--hyperv-virtual-switch` flags |
| `cmd/gokube/cmd/reset.go` | Reset flow | Always-start after restore; Hyper-V lifecycle via `hv.*` |
| `pkg/minikube` | minikube wrapper | `Start` accepts `driver` + `hypervVirtualSwitch`; `DownloadExecutable` accepts bar + caches |
| `pkg/{helm,docker,kubectl,stern,k9s}` | Tool wrappers | `DownloadExecutable` gains bar param + cache check |
| `pkg/{helmspray,helmimage,helmpush}` | Plugin wrappers | `InstallPlugin` gains bar param + cache check + binary path fix |

### Concurrency Model

```go
// Semaphore-based concurrency in UpgradeDependencies
sem := make(chan struct{}, 3)   // cap: 3 concurrent HTTP connections
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

All goroutines complete before any error is returned. This maximises the number of metadata files written on a partial failure — the next retry only downloads what failed.

`UpgradeHelmPlugins` uses the same WaitGroup/Mutex pattern without a semaphore: only three plugins, all run simultaneously.

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

Decision rationale:
- **Per-tool independent files** (not a single JSON manifest): no mutex needed for concurrent writes; each goroutine writes only its own file.
- **Separate `~/.gokube/metadata/` directory** (not sidecar files alongside binaries): keeps the binary directory clean; natural home alongside the existing `config.yaml`.
- **Version string only** (not SHA256): zero maintenance cost per release; SHA256 would require retrieving hashes from six different upstream release pages each version bump.

### Driver Abstraction

```
                 +------------------+
                 |   Hypervisor     |  interface
                 |  (hypervisor.go) |
                 +--------+---------+
                          |
            +-------------+-------------+
            |                           |
  +---------+--------+        +---------+--------+
  |  vboxHypervisor  |        | hypervHypervisor |
  | (virtualbox.go)  |        |   (hyperv.go)    |
  +------------------+        +------------------+
         |                            |
  pkg/virtualbox                  PowerShell
  (VBoxManage)                  (Hyper-V module)
```

The interface is resolved once per command via `hypervisor.New(resolveDriver())`. Adding a third driver (e.g., WSL2) requires: implementing the interface, adding a `case` to `New()`, and handling driver-specific init adaptations in `initRun`. No existing command files need modification.

### Progress Bar Pooling

pb/v3's `StartPool` manages all bars in a single terminal region. Key constraints discovered from reading pb/v3 v3.1.7 source:

- `bar.Start()` spawns an independent render goroutine; calling it before `StartPool` causes each bar to render twice. **Rule: never call `bar.Start()` on a pooled bar.**
- `pb.startTime` is initialised lazily on first `bar.render()` (at pool-start time, before any work begins). `{{etime .}}` therefore reflects pool age, not per-download age.
- `bar.render()` sets `pb.state.time = time.Now()` unconditionally — even for finished bars. The pool keeps ticking until `pool.Stop()`, so a bar's elapsed timer increases until the last goroutine finishes.

Fix: frozen template via named-return defer in `fromUrl`. The deferred closure fires on return; on success it computes elapsed time from a `dlStart` timestamp (set after HTTP headers arrive), writes a static template string, then calls `bar.Finish()`.

---

## Section 5 — Coding Assistant Usage

### Tools Used

| Tool | Role |
|---|---|
| **Claude Code** (claude.ai/code, Sonnet 4.6 — 1M context) | Primary engineering assistant throughout the project |
| **ChatGPT** | Consulted for design discussion and documentation review |

---

### How Claude Code Was Used

**Codebase investigation and call-chain tracing**  
Claude Code was used to trace the full execution path from `gokube init` through `UpgradeDependencies` → `DownloadExecutable` → `download.FromUrl` → `download.fromUrl`. This confirmed that all downloads were sequential, that `DeleteExecutable()` rendered the `os.IsNotExist` guard permanently dead, and that no shared state existed between tool packages — establishing that parallelism was safe to introduce.

**Architecture analysis before implementation**  
Three parallelism approaches (semaphore channel, `errgroup`, mutex-serialised) and three cache storage approaches (sidecar file, `~/.gokube/metadata/`, JSON manifest with checksums) were evaluated collaboratively before writing any code. The recommended approach was argued against gokube's existing architecture and upstream review requirements.

**Library source verification**  
Rather than assuming pb/v3 behaviour from documentation, the actual source files (`pb.go`, `pool.go`, `element.go`) were read from the Go module cache (`/c/Users/.../go/pkg/mod/github.com/cheggaaa/pb/v3@v3.1.7/`). This approach:
- Confirmed that `bar.Start()` unconditionally spawns an independent writer goroutine.
- Identified that `pb.startTime` is initialised lazily at pool-start time, making `{{etime .}}` show pool age rather than per-download elapsed time.
- Confirmed (from `pb.go:466`) that `pb.state.time = time.Now()` is set on every `render()` call even for finished bars — the root cause of the elapsed-time-keeps-growing bug.

**Systematic multi-file refactoring**  
The `*pb.ProgressBar` parameter threading through 14 files (9 tool/plugin packages + download engine + orchestration layer) was executed systematically with parallel edits. Claude Code identified that `helmspray`, `helmimage`, and `helmpush` call `download.FromUrl` directly from `InstallPlugin` — a detail missed in the initial scope pass.

**Integration and conflict analysis**  
Before merging with the Hyper-V branch, Claude Code performed a detailed diff analysis of all overlapping files, identified the 4 files with potential conflicts, and predicted that all 4 would auto-merge based on non-overlapping line ranges. The prediction was correct — zero manual conflict resolutions were required.

**Bug diagnosis**  
Five non-obvious bugs were diagnosed through source-level investigation:
1. Blank `0 [` rows at pool start (no template set before `StartPool`)
2. Duplicate bar output (`bar.Start()` before `StartPool`)
3. Helm plugin cache always missing (wrong binary path in `InstallPlugin`)
4. Inflated elapsed time on cache-hit bars (pool-start `startTime`)
5. Elapsed time growing after 100% (unconditional `state.time = time.Now()` in `render()`)

**Documentation generation**  
`CLAUDE.md`, `docs/hackathon-submission.md`, `docs/hackathon-progress.md`, and this document were written with Claude Code using implementation as the source of truth.

---

### Agentic Features Used

**Claude Code CLI with MCP (Model Context Protocol)** was used throughout. Key agentic capabilities:

- **Multi-file parallel reads and edits** — independent files were processed concurrently in a single response, reducing round-trips.
- **Bash tool execution** — build verification (`go build`, `go vet`), git operations, grep across the module cache.
- **Persistent memory** — a `~/.claude/projects/` memory system was used to preserve architecture decisions, design constraints, and known bugs across sessions without re-reading the full codebase each time.
- **Task tracking** — `TaskCreate`/`TaskUpdate` tools tracked multi-step integration work (branch creation, merge, conflict resolution, build verification, push).
- **GitLab/GitHub MCP** — the teammate's fork was inspected directly via `git ls-remote` and `git fetch` to understand the Hyper-V branch state before integration.

**Key learnings**

- Reading library source from the module cache is more reliable than documentation for understanding subtle API behaviour, particularly around goroutine lifecycle (pb/v3 pool rendering model).
- For integration tasks, a detailed pre-merge conflict analysis (predicting which files will conflict and why) reduces integration time significantly.
- Named-return patterns in Go are underused for deferred cleanup that depends on the success/failure of a function — the `fromUrl` fix is a clean example.

**Limitations observed**

- Claude Code cannot run the binary on the target platform (Windows). All validation was build-level (`go build`, `go vet`) plus manual execution.
- The 1M context window required active memory management across long sessions to avoid losing early architectural decisions.

---

## Section 6 — Quality & Testing

### Build Verification

All changes were verified with a Windows/amd64 cross-compile from the integration host:

```sh
cd cmd/gokube
GOOS=windows GOARCH=amd64 go build ./...   # produces gokube.exe
go vet ./...                                # zero warnings
```

Both commands produced zero errors or warnings across all integration commits.

### Manual Validation

| Scenario | Validation method | Result |
|---|---|---|
| Cold first-run | `gokube init` with no `~/.gokube/metadata/` | All 9 downloads completed; metadata files written; 9 named bars visible |
| Warm re-run | `gokube init` immediately after first run | All 9 bars: `already up to date`; no HTTP requests; total < 1 s |
| Partial failure retry | Kill network mid-download; re-run | Only failed tools re-downloaded; completed tools served from cache |
| Version bump (one tool) | Change one `DEFAULT_*_VERSION`; re-run | Only the modified tool re-downloaded |
| `gokube init -cu` | After a successful first run | All binaries deleted; all metadata deleted; all 9 re-downloaded |
| `gokube init -u` | After a successful first run | Identical to plain init; all cache hits |
| Helm plugin cache | Second run after first install | All three plugins show `already up to date` |
| Progress bar freeze | Download reaching 100% with others still running | Completed bar frozen to `done (Xs)`; other bars continue |
| `gokube reset` (VM stopped) | Reset when VM was not running | Snapshot restored; VM started; `VM was stopped before reset, starting after restore...` |
| Hyper-V `--driver hyperv` | `gokube init --driver hyperv` (elevated shell) | VM created on Hyper-V; driver persisted; subsequent commands use Hyper-V automatically |
| Hyper-V validation — not elevated | `gokube init --driver hyperv` without elevation | Clear error message directing user to run as Administrator |
| VirtualBox default preserved | `gokube init` (no flags) | Identical to pre-hackathon VirtualBox behaviour |

### Regression Testing

- VirtualBox default path was manually verified to behave identically to the pre-hackathon baseline.
- All six VM lifecycle commands (init, start/stop, pause/resume, save/reset) were verified to route through the correct hypervisor implementation.
- The `gokube reset --clean` flag (deletes snapshot after restore) was verified safe: restore completes before `start()` is called.

### Code Quality Improvements

- Removed six unconditional `DeleteExecutable()` calls that preceded each download in `UpgradeDependencies`.
- Removed three unconditional `DeletePlugin()` calls in `UpgradeHelmPlugins`.
- Replaced `addSwapToMinikube()` (hardcoded `/dev/sdb`) with device-node detection (`listMinikubeDisks` / `detectNewSwapDevice` / `formatAndEnableSwap`).
- Removed stale `os/exec` import from `init.go`.
- Added root-level `gokube.exe` to `.gitignore`; removed committed binary from git history.
- Removed the `if !keepVM { if askForUpgrade { ... } }` nesting that made the upgrade path non-obvious.

---

## Section 7 — Technical Documentation

### Architecture

gokube follows a command → orchestration → tool-wrapper → exec layering:

```
main.go
  └── cmd/root.go          (Cobra root; version constants; global vars; resolveDriver)
        ├── cmd/init.go    (init flow: download → VM → chartmuseum → plugins → config)
        ├── cmd/start.go   (minikube start via persisted driver)
        ├── cmd/reset.go   (snapshot restore → always start)
        ├── cmd/save.go    (snapshot create)
        ├── cmd/pause.go
        ├── cmd/resume.go
        └── cmd/swap.go    (swap device detection helpers)

pkg/gokube/gokube.go       (orchestration: UpgradeDependencies, UpgradeHelmPlugins, WriteConfig)
pkg/download/download.go   (HTTP fetch, archive extraction, cache helpers)
pkg/hypervisor/            (Hypervisor interface + VirtualBox and Hyper-V implementations)
pkg/virtualbox/            (VBoxManage wrappers, registry edits, DHCP lease clearing)
pkg/minikube/              (minikube CLI wrappers)
pkg/{helm,docker,kubectl,stern,k9s}/     (tool download + exec wrappers)
pkg/{helmspray,helmimage,helmpush}/       (plugin download wrappers)
pkg/utils/                 (path helpers, archive extraction, progress bar reader)
```

### Execution Flow — `gokube init`

1. Parse flags: `--driver`, `--hyperv-virtual-switch`, `--memory`, `--cpus`, `--kubernetes-version`, etc.
2. `gokube.ReadConfig` — load persisted settings from `~/.gokube/config.yaml`.
3. Version check — if `gokube-version` in config is lower than `GOKUBE_VERSION` (1.38.0), force `askForClean = true`.
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
  os.RemoveAll(VersionFile(localFile))   // ← metadata always purged on miss
  download.FromUrl(url, version, ..., bar)
  return WriteVersion(localFile, version)
```

`WriteVersion` is called only on success. A crash during download leaves no metadata file, so the next run detects a cache miss and retries correctly.

For helm plugins, the binary path is `<pluginDir>/bin/<exe>` — not `<pluginDir>/<exe>`. The split between `pluginDir` (used for `os.RemoveAll`) and `installedBinary` (used for `IsCurrentVersion`/`WriteVersion`) is essential for correct caching.

### Progress Bar Implementation

Three templates cover all bar states:

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

The deferred closure in `fromUrl`:

```go
func fromUrl(...) (n int64, retErr error) {
    // ...
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

`dlStart` is set after HTTP response headers arrive, measuring actual body transfer + archive extraction time.

### Helm Plugin Workflow

```go
func UpgradeHelmPlugins(plugins *HelmPlugins) error {
    // Pre-create 3 bars with waiting-state templates
    pool, _ := pb.StartPool(bars...)

    // Three goroutines, no semaphore
    var wg sync.WaitGroup
    for _, task := range tasks {
        wg.Add(1)
        go func(t func() error) {
            defer wg.Done()
            t()
        }(task)
    }
    wg.Wait()
    pool.Stop()
    return firstErr
}
```

Each `InstallPlugin` follows: `IsCurrentVersion(installedBinary)` → cache hit or `os.RemoveAll(pluginDir)` + `download.FromUrl(pluginDir)` + `WriteVersion(installedBinary)`.

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
2. `Get-VM -ErrorAction Stop` availability check.
3. `Get-VMSwitch -Name '<switch>'` if a name was supplied.

**Config persistence**:

```go
// Written at end of init
viper.Set("minikube-driver", driver)
viper.Set("hyperv-virtual-switch", hypervVirtualSwitch)

// Read by all other commands
func resolveDriver() string {
    if d := viper.GetString("minikube-driver"); len(d) > 0 {
        return d
    }
    if d := os.Getenv("MINIKUBE_DRIVER"); len(d) > 0 {
        return d
    }
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
      root.go          init.go       reset.go      save.go
      start.go         pause.go      resume.go     swap.go
      version.go

pkg/
  gokube/              ← orchestration (UpgradeDependencies, UpgradeHelmPlugins, WriteConfig)
  download/            ← HTTP fetch, archive extraction, cache helpers
  hypervisor/          ← Hypervisor interface + VirtualBox shim + Hyper-V impl
  virtualbox/          ← VBoxManage wrappers, registry, DHCP leases
  minikube/            ← minikube CLI wrappers
  helm/                ← helm CLI wrappers + DownloadExecutable
  docker/              ← docker CLI wrappers + DownloadExecutable
  kubectl/             ← kubectl CLI wrappers + DownloadExecutable
  stern/               ← stern CLI wrappers + DownloadExecutable
  k9s/                 ← k9s CLI wrappers + DownloadExecutable
  helmspray/           ← InstallPlugin (helm-spray)
  helmimage/           ← InstallPlugin (helm-image)
  helmpush/            ← InstallPlugin (helm-push)
  utils/               ← path helpers, archive extraction, progress bar reader

scripts/
  setup-hyperv-switch.ps1    ← PowerShell helper for Hyper-V virtual switch setup

docs/
  hackathon-submission.md
  hackathon-progress.md
  hackathon-4-quadrant.md
  hackathon-presentation.md
  A4I Hackathon_Neural Ninjas_MarkdownFile.md

CLAUDE.md              ← architecture decisions, design constraints, known issues
CHANGELOG.md
README.md
```

### Key Packages Modified

| Package | Key changes |
|---|---|
| `pkg/gokube` | `UpgradeDependencies` rewritten (goroutines + semaphore + pool); `UpgradeHelmPlugins` rewritten (goroutines + pool); `DeleteAllExecutables()` added; `WriteConfig` signature extended |
| `pkg/download` | `fromUrl` named-return defer; `VersionFile`, `IsCurrentVersion`, `WriteVersion`, `DeleteAllMetadata` added |
| `pkg/hypervisor` | Entirely new package (6 files) |
| `pkg/minikube` | `Start` extended with `driver`/`hypervVirtualSwitch`; `DownloadExecutable` extended with bar + cache |
| 5 tool packages | `DownloadExecutable` extended with bar + cache |
| 3 plugin packages | `InstallPlugin` extended with bar + cache; `pluginDir`/`installedBinary` path split |
| `cmd/init.go` | Unconditional checks; clean block; driver flags; hypervisor validation |
| `cmd/reset.go` | Always-start; Hyper-V lifecycle |
| `cmd/pause/resume/save` | Hyper-V lifecycle via `hv.*` |
| `cmd/swap.go` | New — device detection helpers |

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
3. Add `DEFAULT_MINIKUBE_DRIVER` option and handle driver-specific adaptations in `initRun` (IP check, DHCP skip).

**Adjusting download concurrency**: change the single constant `3` in `make(chan struct{}, 3)` in `UpgradeDependencies`.

**Adding a fourth helm plugin**: add a bar to the pool, add a goroutine + task, add the `InstallPlugin` call. The semaphore-free design means no additional code needed unless the count exceeds comfortable simultaneous connections.

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

- `pkg/hypervisor` **must not** import `pkg/minikube`. The driver name and virtual switch flow into `minikube.Start` as plain parameters from the cmd layer.
- `bar.Start()` must **never** be called on a bar that will be passed to `StartPool`. It spawns a competing render goroutine.
- `WriteVersion` must only be called on the **success path** after a completed download. On error, no metadata file is written so the next run retries.
- `DeleteAllExecutables()` must be called with all working directory deletes **before** `upgradeDependencies()`, not after.

### Known Limitations

| Limitation | Notes |
|---|---|
| No automated test suite | gokube has no test infrastructure. All validation is manual. |
| Semaphore cap of 3 not measured | Chosen conservatively for corporate proxies; not benchmarked. |
| `~50%` speedup is estimated | Based on architectural analysis, not measured timing. |
| Swap on Hyper-V is experimental | Documented in README and CHANGELOG. |
| Elapsed time on active downloads is pool-relative | `dlStart` is set after HTTP headers; includes body transfer + extraction. For tools in the first semaphore batch, this is a good approximation. |
| VirtualBox and Hyper-V cannot coexist on the same host | Windows limitation; not a gokube constraint. |

### Future Improvements

| Enhancement | Notes |
|---|---|
| `--download-concurrency N` flag | Expose semaphore size as a CLI flag |
| Download retry on transient failures | Wrap `fromUrl` in a 2–3 attempt retry loop |
| `gokube version --all` from metadata | Read `~/.gokube/metadata/` without exec'ing each binary |
| WSL2 driver support | Pre-existing `TODO` in `root.go`; interface is ready |
| Automatic driver detection | Detect whether Hyper-V is active and suggest `--driver hyperv` |
| JSON manifest with checksums | Enterprise integrity verification; adds per-release maintenance cost |
| Automated test suite | Would require significant infrastructure investment |

### References

- pb/v3 source: `go/pkg/mod/github.com/cheggaaa/pb/v3@v3.1.7/`
- [minikube Hyper-V driver documentation](https://minikube.sigs.k8s.io/docs/drivers/hyperv/)
- [ThalesGroup/gokube](https://github.com/ThalesGroup/gokube)
- [ThalesGroup/miniapps](https://thalesgroup.github.io/miniapps)

---

*Neural Ninjas — A4I Hackathon*
