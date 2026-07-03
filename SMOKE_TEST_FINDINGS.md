# Windows Smoke-Test Findings — 2026-07-02

Real Windows 10 smoke test of build `5ed1293` (release `windows-smoke-test-1`).

## Result summary
- ✅ **Tier 0.1 — WebView2 config window renders AND is interactive.** Validates the critical thread-affinity fix (finding #7 from the QC pass) on real hardware — this was the single biggest runtime risk.
- _(more as testing continues)_

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

### F3 — Installed browsers not detected in rule dropdown · 🟠 Major · DIAGNOSING
**Seen:** the rule browser dropdown is empty (no auto-detected browsers), forcing "Custom...".
**Root cause:** `browser.GetInstalledBrowsers()` returning empty on the box (HKCU+HKLM scan added in 5ed1293, but detection had no logging to confirm what it sees). Could be a registry-read bug or an environment specific.
**Fix:** re-added diagnostic logging to `detect_windows.go` so the next build's logs (`%APPDATA%\Finicky\Logs`) reveal registry-open failures + detected count. Fix once we see the logs.
**Status:** diagnostic added; needs a run on the box to confirm root cause.

### F4 — New rules cannot be created (focus lost between fields) · 🔴 Critical · FIX STAGED
**Seen:** in a new rule, tabbing from the custom-browser input to the URL box fails; clicking from the URL box to the browser box fails. Net: rules can't be saved.
**Root cause:** `Rules.svelte` saves on every field `onblur` → Go echoes a `rules` message → the `$effect` reassigns `rules`, re-rendering the rows **during** the focus transition, so the move to the next field is lost. Deterministic under WebView2's event timing.
**Fix:** added a `selfSaved` guard so the echo of our *own* save no longer clobbers local edit state / re-renders rows (external changes still sync). Preserves focus across field transitions.
**Status:** fix staged in `Rules.svelte`; needs re-test on the box (WebView2 focus behavior can't be verified from macOS).

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

---

_Template for new findings:_
### F# — <short title> · <🔴/🟠/🟡> · <status>
**Seen:** what you observed.
**Root cause:** (filled in after triage).
**Fix:** (plan / done).
**Status:**
