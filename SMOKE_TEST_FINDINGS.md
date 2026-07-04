# Windows Smoke-Test Findings — 2026-07-02

Real Windows 10 smoke test of build `5ed1293` (release `windows-smoke-test-1`).

## Result summary
- ✅ **Tier 0.1 — WebView2 config window renders AND is interactive.** Validates the critical thread-affinity fix (finding #7 from the QC pass) on real hardware — this was the single biggest runtime risk.
- ✅ **Tier 0.2 / 0.3 — resolver + launcher.** `--dry-run` and real launch resolve `https://example.com` → Microsoft Edge and run the correct msedge.exe command (also confirms F1).
- ✅ **Tier 0.5 — single-instance IPC.** After the F8 fix, two launches share one process: the second's URL arrives over the named pipe (`Received URL from IPC`) and routes; no duplicate spawns.
- ✅ **UI↔Go bridge round-trips.** After the F3/bridge fix, `getRules` / `getInstalledBrowsers` reach Go and responses return (was completely dead before — see F3).

_Second on-box pass — 2026-07-03, build `7babaf3` (native Win11 build via winget Go 1.26 / Node 24)._

### Headless verification pass (2026-07-03, build `7babaf3`)
All green — driven via CLI + logs/stderr (no GUI needed):
- ✅ **Tier 0.4 — `finicky://open/<base64>` protocol.** `finicky://open/aHR0…` decoded to `https://protocol-decode-test.example.com/path?x=1` and routed to Edge.
- ✅ **`--config` loading + rule evaluation.** Valid config (`defaultBrowser: Google Chrome`, handler `*example.com* → Microsoft Edge`): `example.com` → Edge (handler), `other.org` → Chrome (default). Confirms handler match, default fallback, and that **goja-babel + esbuild config transform works on Windows** (a cache-miss config triggered a real 247ms babel transform).
- ✅ **`--config` error handling.** Broken-syntax config → precise `SyntaxError … Unexpected token, expected "," (3:2)` with a code frame, then graceful fallback to default routing (no crash). Missing `--config` path → clean "no config file found at <path>" + default routing.
- ✅ **Tier 0.3 — real launch (non-dry-run).** `Run command` executed `msedge.exe <url>` with no error (opened as a tab in already-running Edge).
- ✅ **Profile enumeration (Chromium).** `GetProfilesForBrowser` reads each browser's `Local State`: Edge → 1 profile, Chrome → 3, Firefox/unknown → 0.
- ⚠️ **Note (not a port bug):** Chrome profile list can contain duplicate display names (two profiles both named "Brenna") — `getAllChromiumProfiles` doesn't dedupe, and `parseProfiles` resolves a name to the first match, so a same-named second profile is unreachable by name. This is **shared** behavior (identical logic in macOS `launcher.go`), pre-existing, not Windows-specific — flagged for awareness only.
- **Gotcha for future testers:** a config without `logRequests: true` sets `shouldLog:false`, so **no file appears in `%APPDATA%\Finicky\Logs`** — use `--dry-run` with stderr captured, or add `options.logRequests: true`, when testing configs. (No-config runs default to logging on.)

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
**Status:** ✅ Fixed + verified on box, now including **visual confirmation** (2026-07-04 screenshots): the rule browser dropdown and the Preferences default-browser dropdown both paint **Google Chrome / Internet Explorer / Microsoft Edge** (+ Custom). Post-fix log also shows `getRules` + `getInstalledBrowsers` received by Go with 3 browsers returned.

### F4 — New rules cannot be created (focus lost between fields) · 🔴 Critical · FIX STAGED
**Seen:** in a new rule, tabbing from the custom-browser input to the URL box fails; clicking from the URL box to the browser box fails. Net: rules can't be saved.
**Root cause:** `Rules.svelte` saves on every field `onblur` → Go echoes a `rules` message → the `$effect` reassigns `rules`, re-rendering the rows **during** the focus transition, so the move to the next field is lost. Deterministic under WebView2's event timing.
**Fix:** added a `selfSaved` guard so the echo of our *own* save no longer clobbers local edit state / re-renders rows (external changes still sync). Preserves focus across field transitions.
**Status:** `selfSaved` guard staged in `Rules.svelte`, and **on-box screenshots (2026-07-04) show it working**: multiple rules were built with a browser + 2 URL patterns each, plus a third rule added live, all persisted to `rules.json` ("Config loaded ✓"). Rules with several fields could not have been assembled if focus were lost on each transition, so F4 is effectively confirmed fixed. (Frame-by-frame focus timing wasn't captured, but the functional outcome is verified.) Note: the F3/bridge fix was a prerequisite — before it `saveRules` never reached Go, so nothing could persist regardless of focus.

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

### F9 — Rule patterns silently don't match (glob vs. hostname mental model) · 🟡 UX · NOT-A-PORT-BUG (observed 2026-07-04)
**Seen (on box):** user set rules `Google Chrome → *.google.com` and `Microsoft Edge → test.com`, then Test tab on `https://google.com` returned **Microsoft Edge**, not Chrome. Reported as a bug.
**Root cause — expected finicky behavior, not a Windows defect:** finicky string `match` is a **glob over the whole normalized URL** (`https://google.com/`, scheme + trailing slash), not a hostname match. Verified via the real `finickyConfigAPI` in a resolver repro:
| pattern | `https://google.com` | `https://www.google.com` |
|---|---|---|
| `*.google.com` (their rule) | Edge | Edge |
| `*.google.com/*` | Edge | Chrome |
| `*google.com/*` | Chrome | Chrome |
| `*google.com*` | Chrome | Chrome |
So `*.google.com` fails because (a) no trailing wildcard vs the normalized trailing `/`, and (b) `*.` requires a subdomain dot the apex lacks; `test.com` fails for having no wildcard. The engine + Test tab are working **correctly** — and the fact that the test resolved via the user's rule set proves the whole Windows save→VM-rebuild→resolve→Test pipeline works. Matching is shared JS (`finickyConfigAPI`), identical on macOS.
**Real (shared, not Windows) sharp edge worth a product decision:** the Rules UI's wildcard warning (`patternNeedsWildcard`) only fires when a pattern has **no `*` at all**. A pattern like `*.google.com` has a `*`, so it shows **no** warning, yet still matches nothing — giving false confidence. Options for johnste/Rules-UI: auto-suffix `/*`, treat bare hostnames as hostname matches, or broaden the warning. **Fix for the user right now:** use `*google.com/*` (or `*google.com*`).
**Status:** not a bug; user educated. UX observation logged for the shared Rules editor (defer to upstream).

### Installer + Tier 1/Tier 2 pass (2026-07-03 evening, build `7babaf3`)
**Installer (first run on real hardware) — ✅ full lifecycle verified:**
- Compiled `scripts/installer.iss` with Inno Setup 6 (winget `JRSoftware.InnoSetup`); only a harmless unused-var hint.
- Silent per-user install (`/VERYSILENT /MERGETASKS=autostart`, **no UAC** thanks to `PrivilegesRequired=lowest`): exit 0; exe + uninstaller in `%LOCALAPPDATA%\Finicky`; ProgID `FinickyURL`, `finicky://` protocol, StartMenuInternet + Capabilities, RegisteredApplications, and autostart Run key all written correctly (verified in registry).
- Silent uninstall: exit 0; **every** registry key removed; running process killed. Reinstall: exit 0, all restored.
- ⚠️ Packaging wart: install dir `{localappdata}\Finicky` **collides with the app's cache dir** (`os.UserCacheDir()\Finicky`) — config cache/bundles sit next to the exe, and `[UninstallDelete]` wipes them with the program. Recommend `DefaultDirName={userpf}\Finicky` (= `%LOCALAPPDATA%\Programs\Finicky`) to separate program from cache. Also: uninstall couldn't delete legacy AF_UNIX `.sock` reparse-point files (pre-F8 artifact only; named-pipe builds never create them — non-issue for fresh installs).

**Tier 1 (registration + OS handoff) — ✅ except the human step:**
- `finicky://open/<b64>` via `Start-Process` (real ShellExecute → OS resolves handler): launched the **installed** exe, decoded, routed to Edge. PASS.
- Second protocol launch with primary resident: raw URL handed over the named pipe (`Received URL from IPC`), decoded by primary, routed; still exactly 1 process. PASS (F8 re-verified through the real installed path).
- Startup log correctly reports "Finicky is not the default browser" pre-selection.
- ⏳ Remaining (needs a human): Settings ▸ Default apps ▸ set Finicky (ProgID `FinickyURL`), click http(s) links from real apps, confirm routing + "Finicky is the default browser" log + Settings does NOT reopen per click. Cleanup after: reset default browser; uninstaller removes registration.

**Tier 2 — ✅ profile routing:** rules file with `browser: "Google Chrome", profile: "dan"` → resolver resolves the display name via Local State (`Found profile by name name=dan path=Default`) → dry-run command includes `--profile-directory=Default`. keepRunning residency observed throughout (resident primary routes successive URLs).

**Deliberately tabled (owner decision 2026-07-03):** second-machine validation (esp. Win10 without WebView2 Runtime) — shipping risk accepted for now; revisit before public release if possible.

### F10 — Finicky not selectable as default browser in Win11 Settings · 🔴 Release-blocking · PARTIALLY FIXED (2026-07-03/04)
**Seen:** with the shipped registration (installer or .reg), Settings ▸ Default apps never offers Finicky — not in the HTTPS-link picker, not in the app list. A user could install Finicky and have no way to make it their browser. Diagnosed live on Win11 Home build 26200.
**Root cause — three stacked issues:**
1. **Registration shape too minimal.** Win11's http/https picker filters to apps that look like full browsers: `Capabilities\FileAssociations` (.htm/.html), `Capabilities\Startmenu`, and `InstallInfo` must exist alongside `URLAssociations`. We only shipped URLAssociations.
2. **No association-change notification.** The installer's `[Code]` stub deliberately skipped `SHChangeNotify(SHCNE_ASSOCCHANGED)` as "optional polish" — it is not optional; without it (and sometimes even a shell restart), Settings doesn't re-enumerate.
3. **HKCU alone was not sufficient on this build.** Even with the full browser shape + notifications + `SystemSettings`/Explorer restarts, the picker only listed Finicky after the registration was mirrored to **HKLM** (machine-level — where Chrome/Edge live). ⚠️ Strategic implication: the per-user, no-admin installer (`PrivilegesRequired=lowest`) may be unable to produce a selectable default browser on current Win11. Needs a decision: elevate the installer (admin/dialog) and write HKLM, or keep per-user and accept/UX-around the limitation. (Unclear whether a sign-out would have surfaced the HKCU-only registration — couldn't test without dropping the owner's remote session; a fresh-boot HKCU-only test is the missing data point.)
**Also found (cosmetic but shipped):** the picker labels the entry with the exe's **`FileDescription`** from the PE version resource — which was "Rule-based browser router", not "Finicky". Windows uses FileDescription as a Win32 app's display name (Chrome's is "Google Chrome"). Fixed: `versioninfo.json` FileDescription → "Finicky", `.syso` regenerated, exe rebuilt/redeployed; stale labels purged from shell `MuiCache`.
**Fixed so far:** installer.iss + finicky-register.reg now write the full browser shape; installer.iss `[Code]` now really calls `SHChangeNotify` post-install; FileDescription corrected. Installer recompiles clean.
**Open:** the HKCU-vs-HKLM decision (owner + johnste); HKLM cleanup script (tonight's HKLM mirror on this box has no uninstaller — removal needs an elevated delete of `HKLM\SOFTWARE\Clients\StartMenuInternet\Finicky`, `HKLM\SOFTWARE\Classes\FinickyURL`, and the `HKLM\...\RegisteredApplications` value); fresh-boot HKCU-only retest.
**Status:** Finicky now appears in the Win11 picker (verified by owner screenshot, "New" badge). Default-set + real link-click test in progress.

### F11 — Start Menu shortcut appears dead (no window) · 🟠 Major · ✅ FIXED (2026-07-03)
**Seen (owner report):** launching Finicky from the Start Menu shows nothing — "broken or not rendering."
**Root cause:** the installer's `[Icons]` entry launches `Finicky.exe` with **no arguments**. A bare launch is the windowless resident-router mode (correct for autostart) — and if a primary is already resident, the second instance exits silently after finding nothing to hand off. Reproduced on-box: no-arg launch with resident primary → process count unchanged, no window, no error.
**Fix:** shortcut now passes `--window`. Verified the full path: with a resident primary, `--window` sends the show-window sentinel over the named pipe → primary logs `Received show-window request from another instance` → `Creating window` → window appears. Patched the installed `.lnk` in place and fixed `installer.iss`; installer recompiles clean.
**Status:** ✅ fixed + verified live (window opened via pipe request).

### F12 — Default-browser choice unstable / silently rejected · 🔴 Release-blocking (pending fresh-boot retest) · OPEN
**Seen (Win11 Home 26200):** owner set HTTPS→Finicky via Settings at ~21:48 and it **worked** — four Discord clicks routed through Finicky (`Received URL from IPC` → Edge, 21:49). By 21:55 `UserChoice` read `ChromeHTML` again (silent revert, ~minutes). A second Set-default attempt at ~22:07 **did not take at all**: registry monitored at 2s resolution showed zero change to `UserChoice`, `UserChoiceLatest` absent on this build, no `Microsoft-Windows-Shell-Core/AppDefaults` events logged for either the set, the revert, or the retry.
**Context/confounds:** the registration was mutated repeatedly mid-session (HKCU browser-shape keys added → HKLM mirror added → exe hot-swapped for the FileDescription fix → MuiCache purged). Win11's UserChoice Protection Driver + association hashing may have invalidated trust in the ProgID after those changes — undocumented behavior. This is the wall that led Mozilla to implement the UserChoice hash write directly (Firefox `SetDefaultBrowserUserChoice`).
**Next steps:** (1) **fresh-boot retest** — association state rebuilds at logon; set default once, click once, observe stability (also exercises the installer's autostart task). (2) If instability persists on a clean boot: options are the Firefox-style direct UserChoice+hash write, driving the `ms-settings:defaultapps?registeredAppUser=Finicky` deep link, or documenting a user-facing flow. (3) Retest on a machine that never saw tonight's incremental registration surgery — the F10 second-install test will double as this.
**Status:** OPEN — blocked on reboot (owner is on a remote session; reboot drops it).

### GUI confirmations (2026-07-04 screenshots) — F1 & F6
- **F1** ✅ visually confirmed: Preferences default-browser shows **"System default"** and lists Chrome/IE/Edge — **no "Safari"** anywhere.
- **F6** ✅ visually confirmed: About page reads **"Available on macOS and Windows."**, credits intact (John Sterling, icon @uetchy). Version shows **4.2.2** (the known-stale F7 number).
- **Warning icon renders** under WebView2 (the ⚠️ + "Exact URLs rarely match…" tooltip appeared on the wildcard-less `test.com` row) — rules out a WebView2 SVG-render concern.

---

_Template for new findings:_
### F# — <short title> · <🔴/🟠/🟡> · <status>
**Seen:** what you observed.
**Root cause:** (filled in after triage).
**Fix:** (plan / done).
**Status:**
