# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

gokube is a Windows-only CLI that bootstraps a Kubernetes development environment on a laptop by downloading and orchestrating minikube, docker, helm (+ helm-spray / helm-image / helm-push plugins), kubectl, stern, and k9s, then standing up a VirtualBox-backed minikube VM with ChartMuseum and the [miniapps](https://thalesgroup.github.io/miniapps) helm repo preconfigured. The module path is `github.com/gemalto/gokube`; the published repo is `ThalesGroup/gokube`.

## Build / run

The build target is always `windows/amd64` — the code imports `golang.org/x/sys/windows/registry` (in `pkg/virtualbox`) and shells out to `VBoxManage.exe`, so it cannot be built or run on Linux/macOS natively. CI builds via `GOOS=windows GOARCH=amd64` from Ubuntu.

```sh
# Local Windows build
cd cmd/gokube
go build                                  # produces gokube.exe

# Cross-compile (matches CI)
cd cmd/gokube
GOOS=windows GOARCH=amd64 go build -o bin/gokube-windows-amd64.exe
```

After changing `go.mod` direct dependencies, run from the repo root:

```sh
go get <new-dependency>
go mod tidy
```

There is no test suite (`go test ./...` finds nothing) and no linter configured. Releases are tag-driven: pushing a `v*` tag triggers `.github/workflows/github-release.yaml`, which builds the Windows binary and attaches it to a GitHub release.

## Architecture

**Entrypoint → Cobra commands → tool wrappers → exec.** Everything funnels through `cmd/gokube/main.go` → `cmd/gokube/cmd/root.go` (Cobra `rootCmd`). Each subcommand (`init`, `start`, `stop`, `pause`, `resume`, `save`, `reset`, `version`) is a sibling file under `cmd/gokube/cmd/` that registers itself via its own `init()` and calls into one or more `pkg/<tool>` packages.

**Each external tool has its own package under `pkg/`** (`minikube`, `helm`, `kubectl`, `docker`, `stern`, `k9s`, `helmspray`, `helmimage`, `helmpush`, `virtualbox`). The pattern is identical across them: a `DEFAULT_URL` constant, a `DownloadExecutable` / `DeleteExecutable` pair that uses `pkg/download`, and thin wrappers that build `exec.Command` calls against the external binary. When adding support for a new tool, mirror this layout rather than inventing a new abstraction.

**`DownloadExecutable` now takes a `*pb.ProgressBar` parameter** (added during the hackathon pb/v3 migration). Every caller must supply a bar. For tools called from `UpgradeDependencies` the bar comes from the pre-created 6-bar pool in `gokube.go`. For helm plugins called from `UpgradeHelmPlugins`, bars come from a dedicated 3-bar pool — identical pattern. `InstallPlugin` functions also accept `*pb.ProgressBar`.

**`pkg/gokube/gokube.go` is the orchestration layer** — `UpgradeDependencies` and `UpgradeHelmPlugins` iterate the tool packages and re-download each binary, and `ReadConfig`/`WriteConfig` persist `gokube-version`, `kubernetes-version`, and `container-runtime` to `%USERPROFILE%\.gokube\config.yaml` via viper. The `init` flow in `cmd/gokube/cmd/init.go` is the most important sequence to understand: detect-stale-version → optional clean (with binary + metadata purge if `-c`) → **unconditional** dependency check/download → `minikube delete` → reset VBox host-only DHCP leases → `minikube start` → install ChartMuseum (polls `/index.yaml` on NodePort 32767 to confirm readiness) → add `miniapps` helm repo → patch `kubernetes-dashboard` service to NodePort 30000 → **unconditional** helm plugin check/download → write config.

**`pkg/virtualbox`** is Windows-specific: it edits the registry and parses `VBoxManage list hostonlyifs` / `dhcpservers` output to clear stale DHCP leases that would otherwise prevent minikube from getting the expected `192.168.99.100` IP. This is why most init operations gate on `ipCheckNeeded`.

## Parallel downloads — design decisions

`UpgradeDependencies` in `pkg/gokube/gokube.go` downloads all 6 tool binaries concurrently with a cap of 3 simultaneous downloads. The implementation uses:

- A buffered channel `sem := make(chan struct{}, 3)` as a semaphore — acquiring a slot blocks until one is free.
- `sync.WaitGroup` to block until all 6 goroutines finish.
- `sync.Mutex` + `firstErr error` to capture the first non-nil error race-free; all 6 downloads always run to completion before the error is returned.

This is a deliberate departure from the old sequential fail-fast behavior: all tools are attempted even if one fails, then the first error is surfaced. Callers only check `err != nil` so this is transparent.

**Progress bars use `github.com/cheggaaa/pb/v3` Pool** (`pb.StartPool`). Six bars are pre-created in `UpgradeDependencies` with `pb.New64(0)`. Before calling `StartPool`, each bar gets its template set via `SetTemplateString` with the tool name, version, and `"waiting to start..."`. `bar.Start()` is intentionally **not** called before `StartPool` — doing so spawns an independent render goroutine that conflicts with the pool's goroutine and causes every bar to appear twice. The pool renders bars via `bar.render()` which lazily initialises state on first tick; the pre-set template is rendered immediately. Each goroutine passes its assigned bar through `DownloadExecutable` → `download.FromUrl` → `download.fromUrl`, where the template is replaced with the active download template and the total is set from `Content-Length`. `bar.Start()` is not called in `fromUrl` either. `bar.Finish()` is deferred in `fromUrl` and called explicitly in cache-hit paths. `pool.Stop()` is called after `wg.Wait()`.

**pb/v3 pool rule**: never call `bar.Start()` on a bar that will be passed to `StartPool`. `StartPool` → `pool.Add()` only appends bars; the pool renders them directly via `bar.render()`. Calling `bar.Start()` first creates a second writer goroutine → duplicate terminal output.

**Helm plugins use their own pool**: `UpgradeHelmPlugins` runs all three `InstallPlugin` calls as concurrent goroutines with a dedicated 3-bar `pb.StartPool`. No semaphore is needed — only 3 plugins, all run simultaneously. The pattern (pre-set templates → `StartPool` → goroutines → `wg.Wait()` → `pool.Stop()`) is identical to `UpgradeDependencies`. `InstallPlugin` accepts `*pb.ProgressBar` and handles the cache-hit bar path the same way as `DownloadExecutable`.

## Download cache — design decisions

After a successful download, a metadata file is written to `%USERPROFILE%\.gokube\metadata\<toolname>.version` (e.g. `~/.gokube/metadata/minikube.version` containing `v1.38.0`). On the next `--upgrade` run, `download.IsCurrentVersion(binaryPath, version)` checks both the binary exists and the metadata file content matches the requested version. If so, the download is skipped entirely. The binary directory (`GetBinDir("gokube")`) stays clean — no version files alongside the executables.

Three helpers in `pkg/download/download.go`:
- `VersionFile(binaryPath) string` — derives tool name from `filepath.Base(binaryPath)` (stripping `.exe`), returns `~/.gokube/metadata/<name>.version`
- `IsCurrentVersion(binaryPath, version string) bool` — binary exists + metadata file exists and version matches
- `WriteVersion(binaryPath, version string) error` — ensures `~/.gokube/metadata/` exists via `MkdirAll`, then writes version string

Each `DownloadExecutable` is self-contained: it checks the cache, conditionally deletes and re-downloads, and writes the metadata file on success. The unconditional `DeleteExecutable()` / `DeletePlugin()` calls that previously preceded each download in `UpgradeDependencies` / `UpgradeHelmPlugins` have been removed.

For helm plugins, each `InstallPlugin` uses two separate variables: `pluginDir` (the plugin root) and `installedBinary` (`pluginDir\bin\<exe>`). `IsCurrentVersion` and `WriteVersion` operate on `installedBinary`; `os.RemoveAll` and `download.FromUrl` operate on `pluginDir`. On cache miss, `os.RemoveAll(pluginDir)` removes the plugin directory, and `os.RemoveAll(download.VersionFile(installedBinary))` explicitly removes the metadata file (which now lives in `~/.gokube/metadata/`, outside `pluginDir`).

`DeleteExecutable` calls `os.RemoveAll(download.VersionFile(localFile))` so an explicit forced delete never leaves an orphaned metadata file.

**Cache hit bar behaviour**: when `IsCurrentVersion` returns true, the bar template is updated to `` `{{ green "name" }} version already up to date (<1s)` ``, `SetTotal(1)` / `SetCurrent(1)` are called, and `bar.Finish()` is called. No `bar.Start()` is called (pool already manages the bar). The static string `<1s` is used instead of `{{etime .}}` — see the "etime bug" note below. The active-download template in `fromUrl` retains `{{etime .}}` for real downloads.

**Why cache-hit bars must not use `{{etime .}}`**: `pb.startTime` is initialised lazily on the first `bar.render()` call from the pool — at `StartPool` time, before any goroutine starts work. `{{etime .}}` computes `state.Time().Sub(pb.startTime)` = current render time minus pool-start time. A tool that waits 20 s for a semaphore slot before getting a cache hit would display `already up to date (20s)` despite the check itself taking < 1 ms. There is no public API to reset `pb.startTime` without calling `bar.Start()`, which spawns a competing render goroutine. The fix is to embed the display string as a static literal `(<1s)` in the template, bypassing `{{etime .}}` entirely.

## Version bumps

A single release typically bumps several constants together — failing to update them in lockstep causes the `checkMinimumRequirements()` guard or the persisted-version comparison in `initRun` to misbehave:

- `GOKUBE_VERSION` in `cmd/gokube/cmd/version.go` — drives the "force clean & upgrade on first run" path in `initRun` (compared via `semver` against the value previously written to `~/.gokube/config.yaml`). Current value: `1.38.0`.
- `DEFAULT_*_VERSION` constants in `cmd/gokube/cmd/root.go` — defaults for kubernetes (`v1.35.0`), kubectl (`v1.35.0`), minikube (`v1.38.0`), docker (`29.2.1`), helm (`v3.20.0`), helm-spray (`v4.0.13`), helm-image (`v1.1.0`), helm-push (`0.10.4`), stern (`1.33.1`), k9s (`0.50.18`).
- Add a `CHANGELOG.md` entry at the top.
- Minimum-version floors live in `checkMinimumRequirements()` in `root.go` — only touch these when intentionally raising the floor.

## init command semantics

Three command variants have distinct behaviors, all sharing the same cache-aware download functions:

| Command | What happens |
|---|---|
| `gokube init` | `upgradeDependencies()` + `upgradeHelmPlugins()` always run unconditionally. Cache hits complete in < 1 s. Only missing or version-mismatched tools are re-downloaded. Fast common path. |
| `gokube init -u` | Identical code path. The `-u` flag is kept for backward compatibility but has no additional effect since the check always runs. |
| `gokube init -cu` | Clean block runs first: `DeleteWorkingDirectory()` for all tools (runtime state), then `gokube.DeleteAllExecutables()` (removes binaries + all metadata in `~/.gokube/metadata/`). With everything wiped, `IsCurrentVersion` returns false for all 9 tools → full re-download. Equivalent to a fresh machine. |

**Key functions:**
- `gokube.DeleteAllExecutables()` in `pkg/gokube/gokube.go` — calls `DeleteExecutable()` for all 6 main tools then `download.DeleteAllMetadata()`. Called from `initRun` when `askForClean=true`.
- `download.DeleteAllMetadata()` in `pkg/download/download.go` — `os.RemoveAll(~/.gokube/metadata/)`. Wipes all version files in one call.

**The `-u` flag**: `upgradeDependencies()` and `upgradeHelmPlugins()` are unconditional in `initRun`. The `askForUpgrade` variable is still set (for backward compat and the first-run forced-clean path) but no longer gates the download calls.

## Configuration sources

Most settings have three layers, resolved in this order: CLI flag → env var (`utils.GetValueFromEnv` in `pkg/utils/utils.go`) → constant default. The persisted `~/.gokube/config.yaml` only stores `gokube-version`, `kubernetes-version`, and `container-runtime` — it is used by `start`/`stop`/etc. to remember what `init` was run with, not as a general settings file.

Proxy support (`HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`) is wired through to both the downloader and the minikube docker daemon (`--docker-env=...` flags built in `pkg/minikube/minikube.go::Start`).

## Key files and responsibilities

| File | Responsibility |
|---|---|
| `cmd/gokube/cmd/root.go` | Default version constants, `upgradeDependencies()` / `upgradeHelmPlugins()` wrappers, `checkMinimumRequirements()` |
| `cmd/gokube/cmd/init.go` | Full `init` flow including VM lifecycle, ChartMuseum, dashboard, swap |
| `cmd/gokube/cmd/version.go` | `GOKUBE_VERSION` constant, `version` command |
| `pkg/gokube/gokube.go` | `UpgradeDependencies` (parallel, pooled bars), `UpgradeHelmPlugins` (parallel, own pool), `DeleteAllExecutables()`, `ReadConfig`/`WriteConfig` |
| `pkg/download/download.go` | `FromUrl` / `fromUrl` — HTTP fetch, archive extraction, progress bar rendering; `VersionFile` / `IsCurrentVersion` / `WriteVersion` / `DeleteAllMetadata` — cache helpers |
| `pkg/utils/utils.go` | `ClosePBReader`, archive helpers (`Untar`, `Unzip`), path helpers |
| `pkg/virtualbox/` | Windows registry edits, DHCP lease clearing via `VBoxManage` |
| `pkg/minikube/minikube.go` | `Start`, `Delete`, `Ip`, `ConfigSet`, `AddonsEnable` wrappers |

## Development workflow

1. Make changes in `pkg/` or `cmd/gokube/cmd/`.
2. Build: `cd cmd/gokube && go build` — confirms the code compiles for `windows/amd64`.
3. Test manually: run `./gokube.exe init` — VirtualBox GUI must be closed, no other VM running.
4. If `go.mod` changed: run `go mod tidy` from repo root before building.
5. Commit and push a `v*` tag to trigger the GitHub release workflow.

## Current project status (as of end of session 2)

All hackathon work is complete and build-verified (`go build` + `go vet` both pass). **Not yet committed.** The working tree has modified files across `pkg/`, `cmd/gokube/cmd/init.go`, `.gitignore`, and all docs.

**What to do before the next session:**
```sh
git add go.mod go.sum pkg/ cmd/ .gitignore CLAUDE.md docs/
git commit -m "Parallel downloads, pb/v3 pool, download cache, helm plugin pool, init semantics fix, elapsed time bars"
```

**Session 1 — what was implemented:**
- Parallel downloads for 6 main tools (semaphore 3, WaitGroup, Mutex)
- pb/v2 → pb/v3 migration with 6-bar pool rendering
- Named "waiting to start..." bars for all 6 tools
- Download cache: `~/.gokube/metadata/` version files prevent re-downloads
- Four bugs found and fixed (see `docs/hackathon-progress.md`)

**Session 2 — what was additionally implemented:**
- Helm plugin parallelization: `UpgradeHelmPlugins` now uses its own 3-bar pool + goroutines (no semaphore)
- `InstallPlugin` signature updated to accept `*pb.ProgressBar` with cache-hit bar handling
- `gokube init` / `init -u` / `init -cu` semantics corrected:
  - `upgradeDependencies()` and `upgradeHelmPlugins()` are now unconditional in `initRun`
  - `init -cu` now purges binaries + all metadata via `DeleteAllExecutables()` before downloading
  - `DeleteAllExecutables()` added to `pkg/gokube/gokube.go`
  - `DeleteAllMetadata()` added to `pkg/download/download.go`
- Elapsed time (`{{etime .}}`) added to all 9 tool/plugin cache-hit templates and the active-download template
- `.gitignore` updated: `cmd/gokube/gokube.exe` and `cmd/gokube/go` added

## Architecture discoveries

**`gokube init` triggers forced upgrade on first run** (`cmd/gokube/cmd/init.go:236–247`): if `~/.gokube/config.yaml` is absent or the stored `gokube-version` is lower than `GOKUBE_VERSION` (currently `1.38.0`), both `askForClean` and `askForUpgrade` are forced true. This is why `gokube init` (without `--upgrade`) still runs `UpgradeDependencies` on a fresh machine.

**`--clean` does not touch binaries**: `DeleteWorkingDirectory` functions clear runtime state (`~/.minikube`, `~/.kube`, `~/.docker`, `%APPDATA%\helm`) but never `GetBinDir("gokube")` or `~/.gokube/metadata/`. After `--clean`, the 6 main tool metadata files survive; helm plugin metadata files also survive but their binaries are gone (plugin dirs deleted as part of `%APPDATA%\helm`), so `IsCurrentVersion` correctly detects a cache miss and re-downloads them.

**Helm plugins use `bin/<exe>` not `<plugin>/<exe>`**: The `localFile` path used for existence checks in `InstallPlugin` pointed to the plugin root, but all three plugins (helmspray, helmimage, helmpush) install their binaries under a `bin/` subdirectory via the fileMap. This was dead code in the original (masked by unconditional `DeletePlugin()`), but became a real bug once caching was introduced.

**pb/v3 pool API (verified from source)**:
- `pb.StartPool(bars...)` does **not** call `bar.Start()` internally — it only appends bars to the pool's slice
- `bar.render()` lazily initialises `pb.state` on first call — templates pre-set before `StartPool` render immediately
- `bar.Start()` spawns an independent writer goroutine — calling it before `StartPool` causes duplicate output
- `bar.Finish()` works correctly with nil `pb.finish` channel (no `bar.Start()` needed) — just sets `pb.finished = true`

## Important decisions

**Semaphore size 3 over `errgroup`**: `golang.org/x/sync` is not in `go.mod`. Semaphore channel achieves the same cap with zero new dependencies. Size 3 balances throughput against corporate proxy limits; trivial to change.

**No semaphore for helm plugins**: Only 3 plugins; all run simultaneously. Semaphore would add complexity with no benefit. If a 4th plugin is added, reconsider.

**All-complete-before-error**: All goroutines run to completion even if one fails. First error returned after `wg.Wait()`. Users benefit from having as many metadata files written as possible on a failed run — retry is faster.

**`~/.gokube/metadata/` over sidecar files**: Keeps the binary directory clean. Each tool still has its own independent file (no mutex needed). Three alternatives evaluated: sidecar-alongside-binary (rejected: clutters binary dir), JSON manifest (rejected: requires mutex + concurrent write coordination), SHA256 verification (rejected: very high maintenance cost). Metadata directory is created by `WriteVersion` via `MkdirAll` on first use.

**`DeleteAllMetadata()` for clean purge**: Removing the entire `~/.gokube/metadata/` directory in one `os.RemoveAll` call is simpler and more complete than calling each tool's `DeleteExecutable()` for the metadata side-effect. `DeleteAllExecutables()` does both: deletes binaries (via `DeleteExecutable()`) then wipes the metadata directory (via `DeleteAllMetadata()`).

**`upgradeDependencies()` unconditional**: Making the download check always run (not gated on `-u`) makes `gokube init` self-healing — missing binaries are re-downloaded automatically. The cache means the common case (all tools present at correct version) completes in < 1 s. The `-u` flag is redundant but kept for backward compatibility.

## Known issues and open items

- ~~**Helm plugins not in pool**~~: resolved — plugins now use a dedicated 3-bar pool with named waiting states and run concurrently.
- ~~**`cmd/gokube/gokube.exe` and `cmd/gokube/go` untracked**~~: resolved — both added to `.gitignore`.
- ~~**Inflated etime on cache-hit bars**~~: resolved — cache-hit templates now use static `(<1s)` instead of `{{etime .}}`. See "Why cache-hit bars must not use `{{etime .}}`" above.
- ~~**`gokube reset` leaves VM stopped when VM was already stopped before reset**~~: resolved — `resetRun` now always calls `start()` after a successful restore, with a state-aware message indicating whether it is restarting or starting from stopped.
- **No wall-clock measurement**: parallel speedup (~50%) is estimated; not yet measured with timing on real hardware. Active downloads show `{{etime .}}` (pool-relative, not per-download, but proportional for first-batch tools).
- **No version bump**: all changes are internal. `GOKUBE_VERSION` stays at `1.38.0`. No `CHANGELOG.md` entry needed.
- **`-u` flag is now redundant**: `upgradeDependencies()` always runs. Flag kept for backward compat. Could be documented as deprecated or repurposed in a future release.
- **No automated tests**: gokube has no test suite. All validation is manual via `gokube init`.
