# Windows Smoke-Test Findings — 2026-07-02

Real Windows 10 smoke test of build `5ed1293` (release `windows-smoke-test-1`).

## Result summary
- ✅ **Tier 0.1 — WebView2 config window renders AND is interactive.** Validates the critical thread-affinity fix (finding #7 from the QC pass) on real hardware — this was the single biggest runtime risk.
- ✅ **Tier 0.2 / 0.3 — resolver + launcher.** `--dry-run` and real launch resolve `https://example.com` → Microsoft Edge and run the correct msedge.exe command (also confirms F1).
- ✅ **Tier 0.5 — single-instance IPC.** After the F8 fix, two launches share one process: the second's URL arrives over the named pipe (`Received URL from IPC`) and routes; no duplicate spawns.
- ✅ **UI↔Go bridge round-trips.** After the F3/bridge fix, `getRules` / `getInstalledBrowsers` reach Go and responses return (was completely dead before — see F3).

_Second on-box pass — 2026-07-03, build `e83dad4`+fixes (native Win11 build via winget Go 1.26 / Node 24)._

## Findings

### F1 — Default browser shows "Safari" (should be Edge on Windows) · 🟠 Major · FIX STAGED
**Seen:** the config UI pre-fills / shows the default browser as "Safari", which doesn't exist on Windows.
**Root cause:** three macOS-hardcoded sites:
- `apps/finicky/src/resolver/resolver.go` — `defaultBrowserConfig` → `"com.apple.Safari"` (runtime routing fallback when no config)
- `apps/finicky/src/rules/rules.go` — `ToJSConfigScript` → `"com.apple.Safari"` (rules→JS default)
- `packages/finicky-ui/src/pages/StartPage.svelte` — `const SAFARI = "Safari"` (UI seed — **this is the visible one**)
**Fix:** new platform default `browser.DefaultBrowserName` (build-tagged: `com.apple.Safari` on darwin, `Microsoft Edge` on windows; Edge ships on every Win10/11).
- Go sites (resolver, rules): ✅ FIXED on `windows-support` (uncommitted in working tree); Windows+macOS build + resolver/rules tests green.
- UI site (StartPage.svelte): ⏳ PENDING — needs a UI rebuild. Plan: backend sends `browser.DefaultBrowserName` to the UI so StartPage seeds from it instead of the hardcoded `"Safari"` (keeps macOS showing Safari, Windows showing Edge — no divergence).
**Status:** Go done; UI change + rebuild to be batched with the rest of this smoke round.

### F2 — Test URL: poor error/timeout handling; shows Safari · 🟡 Minor · PARTIAL
**Seen:** the Test URL feature appears to time out or resolve to a non-existent browser (Safari); no clear error state on failure.
**Root cause:** the "Safari" result is F1 (default browser). Beyond that, the UI has no timeout/failure state for the test-url round-trip.
**Fix:** F1 fix makes it report Edge. Add a UI timeout + explicit error rendering for the test result (batched with UI rebuild).
**Status:** Safari part fixed via F1 (Go); UI error-handling pending.

### F3 — Installed browsers not detected in rule dropdown · 🟠 Major · ✅ ROOT-CAUSED + FIXED (2026-07-03)
**Seen:** the rule browser dropdown is empty (no auto-detected browsers), forcing "Custom...".
**Root cause (NOT the registry):** detection works fine — `GetInstalledBrowsers()` returns 3 on this box (Google Chrome, Internet Explorer, Microsoft Edge); logs confirm `HKLM found=3`, `Installed browsers detected count=3`. The real bug was one layer up: **the entire UI→Go message bridge was dead on Windows.** `App.svelte` replaces `window.finicky` after load with a `sendMessage` that posts only to `window.webkit.messageHandlers.finicky` (the macOS WKWebView bridge). On WebView2 `window.webkit` is undefined, so optional-chaining made `sendMessage` a silent no-op — every `getInstalledBrowsers` / `getRules` / `saveRules` / `testUrl` request was dropped before reaching Go (verified: no `Received message from webview` log lines at all pre-fix). This also explains F2 (test URL never returns) and the save half of F4.
**Fix:** `App.svelte` `sendMessage` now routes to whichever bridge exists — `window.webkit.messageHandlers.finicky` on macOS, else `window.__finicky_send` (the WebView2 binding the Windows host already exposes in `window_windows.go`). Added `__finicky_send?` to the `Window` type in `types.ts`. macOS path unchanged (webkit tried first) — no divergence.
**Status:** ✅ Fixed + verified on box. Post-fix log shows `getRules` + `getInstalledBrowsers` received by Go and 3 browsers sent back. (Visual confirmation that the dropdown paints the 3 entries still wants a GUI pass — but the data now round-trips.)

### F4 — New rules cannot be created (focus lost between fields) · 🔴 Critical · FIX STAGED
**Seen:** in a new rule, tabbing from the custom-browser input to the URL box fails; clicking from the URL box to the browser box fails. Net: rules can't be saved.
**Root cause:** `Rules.svelte` saves on every field `onblur` → Go echoes a `rules` message → the `$effect` reassigns `rules`, re-rendering the rows **during** the focus transition, so the move to the next field is lost. Deterministic under WebView2's event timing.
**Fix:** added a `selfSaved` guard so the echo of our *own* save no longer clobbers local edit state / re-renders rows (external changes still sync). Preserves focus across field transitions.
**Status:** `selfSaved` guard staged in `Rules.svelte`. ⚠️ Note: the F3/bridge fix is a prerequisite — before it, `saveRules` never reached Go at all (silent no-op), so no rule could persist regardless of focus. With the bridge alive, the focus behavior itself still needs a GUI pass to confirm (WebView2 focus timing can't be checked from macOS, and this session has no desktop computer-use to drive the native window). Remaining manual verification: open `--window`, add a rule, tab browser↔URL, confirm focus holds and the rule persists on reopen.

### F5 — No Windows icon on the .exe / window · 🟡 Fit-and-finish · FIX STAGED
**Seen:** Finicky.exe has the generic Windows icon (no Finicky icon in Explorer/taskbar/window).
**Root cause:** cross-compiled Go binaries embed no PE resource; the `.ico` existed but was never linked in.
**Fix:** added `versioninfo.json` + generated `resource_windows_amd64.syso` (via goversioninfo) embedding `finicky.ico` + version metadata. Auto-linked for windows/amd64 only (macOS unaffected). Rebuilt exe.
**Status:** staged; embeds on build. (TODO for PR: regenerate `.syso` in CI/build-windows.sh rather than committing the binary.)

### F6 — About page says "macOS application"; author routing · 🟡 Fit-and-finish · PARTIAL
**Seen:** About page reads "Finicky is a macOS application"; credits/links point to johnste, which may route Windows bugs to John.
**Fix (now):** corrected wording to "Available on macOS and Windows." Left author attribution intact — correct for the upstream PR path and MIT-required.
**Convention:** upstream → keep John as author, route Windows issues via maintainer role + `platform:windows` label. Fork → dual attribution ("based on Finicky by John Sterling", Windows port by <maintainer>) and repoint "View on GitHub" to the fork. Fork variant deferred until the upstream/fork decision.
**Status:** wording fixed; attribution intentionally unchanged pending fork decision.

### F7 — Version handling broken on Windows; wrong version labels · 🟠 Major · PARTIAL (Windows `defaults` error fixed 2026-07-03; version *number* still stale)
**Seen:** (surfaced comparing to the public release) About page shows "dev"; update-check sends an empty version. On Windows, three `level=ERROR` lines per startup: `Error reading version from Info.plist … exec: "defaults": executable file not found in %PATH%`.
**Root cause:** `version/version.go` `GetCurrentVersion()` was shared (no build tag) and ran the macOS command `defaults read <Info.plist> CFBundleVersion` — `defaults` does not exist on Windows, so it errored → returned ""/"dev". Separately, `installer.iss` / winget / `versioninfo.json` all hardcode **4.2.2**, but the fork base is **v4.4.0-alpha + 9 commits** (`git describe` = `v4.4.0-alpha-9-gd4d62f8`). 4.2.2 is a stale stable that predates the Rules UI.
**Fix — part 1 (DONE, Windows side, 2026-07-03):** split into platform files. `version_darwin.go` keeps the Info.plist/`defaults` path (unchanged). New `version_windows.go` reads the version from the exe's embedded PE version resource (the same `versioninfo.json` from F5) via `golang.org/x/sys/windows` `GetFileVersionInfo`/`VerQueryValue`. Shared `version.go` loses the macOS body + the now-unused `os/exec` import. Result: **zero `defaults`/Info.plist errors on startup**; version now reads a real number from the resource.
**Fix — part 2 (STILL TODO, Mac side):** the resource read now returns **4.2.2 — the stale hardcoded number**, confirming the second half of the root cause. Correct number still needs the planned `var version string` in `version.go` set via `-ldflags "-X finicky/version.version=…"` (from `git describe`) in build-windows.sh / CI, plus aligning installer/winget/versioninfo to the real base (~4.4.0-alpha; final scheme is johnste's call). The Windows `version_windows.go` should prefer that injected `version` when present and fall back to the PE resource. Left for the Mac side as agreed (shared `version.go`, macOS-sensitive).
**Status:** Windows `defaults`-error half ✅ fixed + verified (`Starting Finicky version=4.2.2`, no errors). Correct-version-number mechanism still triaged/pending on the Mac side.

> **Reference build note:** the macOS **"Latest" release (v4.2.2, Oct 2025) predates the Rules UI** and is NOT a valid reference for F3/F4. The Rules editor arrived in **v4.4.0-alpha**; our fork base is 9 commits past it. Correct reference = v4.4.0-alpha (downloaded) or a current-code macOS build. Architectural read: **F4 (focus) is shared Svelte code** — the `selfSaved` fix helps both platforms. ⚠️ **Correction (2026-07-03, on box):** the earlier read that "F3 (empty browser dropdown) is Windows-specific — `detect_windows.go` returning empty" is **wrong**. On-box, `GetInstalledBrowsers()` returns 3 (Chrome/IE/Edge); the dropdown was empty because the whole UI→Go bridge was dead (see F3). Detection was never the problem.

### F8 — Single-instance IPC unreliable on Windows (AF_UNIX) · 🔴 Critical · ✅ FIXED (2026-07-03)
**Seen:** every startup logs `Failed to start URL listener … listen unix …\finicky.sock: bind: An invalid argument was supplied.` (WSAEINVAL). With the listener dead, single-instance handoff / `keepRunning` routing (Tier 0.5) silently breaks — a second launch can't reach the primary.
**Root cause:** the IPC used a Unix-domain socket (`net.Listen("unix", …)`). Go's AF_UNIX support on Windows is unreliable: a standalone probe listened OK on a truly-fresh path once, then failed with WSAEINVAL on *every* subsequent bind (even fresh names) after a few socket create/remove cycles. AF_UNIX sockets also leave a reparse-point file behind on abnormal exit, and re-binding an existing path fails with WSAEINVAL rather than EADDRINUSE, defeating the remove-and-retry. This is exactly why named pipes — not AF_UNIX — are the correct Windows IPC primitive.
**Fix:** `main_windows.go` now uses a Windows named pipe via `github.com/Microsoft/go-winio` — `winio.ListenPipe(\\.\pipe\FinickyBrowserRouter)` / `winio.DialPipe(...)`. Both return the same `net.Listener`/`net.Conn` interfaces, so `listenForURLs` / `sendToPrimary` bodies barely change. No on-disk artifact → no stale-file handling. Pipe name is machine-wide to match the existing `Global\` single-instance mutex. macOS untouched (Windows-only file).
**New dependency:** `github.com/Microsoft/go-winio v0.6.2` (direct). Battle-tested (Docker/containerd). ⚠️ **Mac session: please ratify this dependency before promoting to `windows-support`/PR #542.**
**Status:** ✅ Fixed + verified end-to-end. `IPC listener started address=\\.\pipe\FinickyBrowserRouter`; two launches → one process; secondary URL delivered via `Received URL from IPC` and routed to Edge; secondary exits 0, no duplicate.

---

_Template for new findings:_
### F# — <short title> · <🔴/🟠/🟡> · <status>
**Seen:** what you observed.
**Root cause:** (filled in after triage).
**Fix:** (plan / done).
**Status:**
