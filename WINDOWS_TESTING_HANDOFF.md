# Windows Testing Handoff

You are a Claude Code instance running **natively on Oz's Windows 10 machine**. Your job: smoke-test the Finicky Windows port first-hand, find and fix bugs, and push fixes back. A parallel Claude session on a Mac cross-compiles and holds the broader project context; **this git fork is the shared source of truth.**

## What Finicky is
A rule-based browser router: you click a link anywhere, Finicky decides which browser (and profile) opens it, based on your rules. Originally macOS-only (Swift → v4 rewrite in Go/Svelte/TS). This is the **Windows port** — Go with `//go:build windows` platform files, a WebView2 config UI via `github.com/jchv/go-webview2` (pure Go, no cgo), registry-based browser detection, and an Inno Setup installer.

## Repo / branch state
- Repo: `github.com/chaoz23/finicky` (a fork of `johnste/finicky`). Upstream draft PR: **johnste/finicky#542**.
- **You are on branch `windows-smoke`** — the active testing/fixing branch. Push your fixes here.
- The **PR branch is `windows-support`** (kept clean). Do NOT commit testing docs (`SMOKE_TEST*.md`, this file) to `windows-support`. Verified *code* fixes get promoted from `windows-smoke` → `windows-support` later (by you or the Mac session, via cherry-pick).
- Read `SMOKE_TEST_FINDINGS.md` (findings F1–F6) and `SMOKE_TEST.md` (the tiered runbook) first.

## Validated so far (on real Win10)
- ✅ WebView2 config window **renders and is interactive** (the critical thread-affinity fix works on hardware).
- ✅ CLI routing works: `Finicky.exe https://google.com` opens the right browser.

## Open / needs your eyes
- **F3** — installed browsers not showing in the rule dropdown. Diagnostic logging was added; check `%APPDATA%\Finicky\Logs` for `Browser registry scan root=HKCU found=…`, `root=HKLM found=…`, `Installed browsers detected count=…`. Determine: is detection returning 0 (registry bug) or are browsers found but not wired to the UI?
- **F4** — creating a new rule: does focus hold when tabbing/clicking between the browser field and the URL field? (Fix staged: `Rules.svelte` `selfSaved` guard. Verify it actually works under WebView2.)
- Hunt for **new** bugs: test URL page, browser profiles, keepRunning, single-instance (two quick launches → one process), `finicky://` protocol, default-app routing (click links from other apps), window resize/close/reopen, config file (`--config`) loading, error states.

## Prerequisites (native Windows build)
- **Go 1.24+**, **Node 22+**. Optional: `goversioninfo` (for the icon resource; already committed as `apps/finicky/src/resource_windows_amd64.syso`, regenerate only if the icon changes).
- **WebView2 Runtime** — required for the config window (Win10 often lacks it). Confirm installed (Settings ▸ Apps ▸ "Microsoft Edge WebView2 Runtime") before blaming a blank window on code.

## Build
```powershell
# 1. Build UI assets (only when you change packages/finicky-ui or config-api)
cd packages/config-api;  npm ci; npm run build; npm run generate-types
copy /Y dist\finickyConfigAPI.js ..\..\apps\finicky\src\assets\finickyConfigAPI.js
cd ..\finicky-ui;        npm ci; npm run build
if not exist ..\..\apps\finicky\src\assets\templates mkdir ..\..\apps\finicky\src\assets\templates
xcopy /E /Y dist\* ..\..\apps\finicky\src\assets\templates\

# 2. Build the exe
cd ..\..\apps\finicky\src
$env:CGO_ENABLED="0"
go build -ldflags "-H windowsgui -X 'finicky/version.commitHash=$(git rev-parse --short HEAD)'" -o ..\build\windows\Finicky.exe .
```
`go test ./...` should pass. `go vet ./...` clean.

## Run / test
```
apps\finicky\build\windows\Finicky.exe --window                 # config UI (F4 lives here)
apps\finicky\build\windows\Finicky.exe https://example.com      # route a URL
apps\finicky\build\windows\Finicky.exe --dry-run https://x.com   # resolve, don't launch
```
- Logs: `%APPDATA%\Finicky\Logs` (JSON, debug level).
- Default-browser registration (Tier 1): double-click `scripts\finicky-register.reg` (assumes exe at `C:\Finicky\` — edit paths if elsewhere), set Finicky in Settings ▸ Default apps, click links from other apps. Clean up with `scripts\finicky-unregister.reg`.
- Enable WebView2 devtools for debugging: set env `FINICKY_DEBUG=1` before launching `--window`.

## Using computer use (for the GUI bugs)
The focus/render bugs (F4, rendering) can only be verified by driving the window: screenshot → click a field → type → Tab → screenshot again. Use computer use to open `--window`, add a rule, and confirm focus moves browser↔URL and a rule actually saves (check the log / re-open window). Report exactly what focus does on failure.

## Recording + pushing
- Append new findings to `SMOKE_TEST_FINDINGS.md` as F7, F8, … (seen / root cause / fix / status).
- Commit code fixes on `windows-smoke`, `git push origin windows-smoke` (origin = the fork). Rebuild and re-verify before claiming a fix works.
- Keep macOS intact: never diverge a `_windows.go` file's behavior from the shared reference (`main.go`/`window.go`) without noting it — this is a port, not a rewrite.

## Safety on a real machine
- The `.reg` files touch `HKCU` only (per-user, no admin). Always run `finicky-unregister.reg` to clean up, and reset the default browser in Settings when done.
- Don't set Finicky as default until you can revert it. Don't push to `windows-support` or interact with the upstream PR (#542) — that's coordinated with the Mac session.
