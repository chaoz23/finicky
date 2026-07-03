# Finicky for Windows — Smoke Test Runbook

Real-hardware validation before flipping [PR #542](https://github.com/johnste/finicky/pull/542) out of draft. Target: **Windows 10** (the higher-risk target — WebView2 runtime is often absent, unlike Win11).

Work top-down. **Tier 0 needs no installation or registration** and validates the fixes most likely to be broken. Only proceed to Tier 1/2 once Tier 0 is green.

---

## Prerequisite: WebView2 Runtime (do this first)

`go-webview2` embeds the *loader* but still needs the **Evergreen WebView2 Runtime** installed. Windows 11 ships it; **Windows 10 frequently does not** — and without it, `--window` opens a blank window (which looks exactly like the thread-affinity bug we fixed, so rule it out first).

1. Check: Settings → Apps → look for "Microsoft Edge WebView2 Runtime", or registry `HKLM\SOFTWARE\WOW6432Node\Microsoft\EdgeUpdate\Clients\{F3017226-FE2A-4295-8BDF-00C3A9A7E4C5}`.
2. If missing, install the **Evergreen Standalone Installer** from Microsoft's WebView2 download page, then re-test.

> If the config window is blank **after** the runtime is confirmed installed → that's a real thread-affinity regression, not a prerequisite issue. Capture it.

## Getting the binary onto the machine

**Download** on the Win10 box from the draft release (sign in as `chaoz23`):
<https://github.com/chaoz23/finicky/releases> → "Windows smoke-test build 1".
It bundles `Finicky.exe` + `finicky-register.reg` + `finicky-unregister.reg`.

Put `Finicky.exe` at **`C:\Finicky\`** — that's the path the `.reg` files assume. (Source of truth on the Mac: `~/Projects/finicky/apps/finicky/build/windows/Finicky.exe`.)

A test config helps for rule-based checks — copy `~/Projects/finicky/example-config/` as a starting point and point Finicky at it with `--config`.

---

## Tier 0 — Core (no registration required)

These exercise the WebView2 threading fix, the resolver, the launcher, and the single-instance IPC directly via CLI args.

| # | Command / action | Pass criteria | Fix it validates |
|---|---|---|---|
| 0.1 | `Finicky.exe --window` | A config UI **renders and is interactive** (click tabs, type in the URL tester). Not blank, not frozen. | **#7 thread-affinity** (the critical one) |
| 0.2 | `Finicky.exe --dry-run https://example.com` | Logs show a resolved browser; no browser launches | resolver + browsers.json on Windows |
| 0.3 | `Finicky.exe https://example.com` | The correct browser actually opens | `launcher_windows.go` (findBrowserExe) |
| 0.4 | `Finicky.exe "finicky://open/<base64-url>"` | Decodes and routes to the underlying URL | protocol decode |
| 0.5 | Run `Finicky.exe https://a.com` then quickly `Finicky.exe https://b.com` (with keepRunning, or within ~2s) | **One** process handles both; second logs "Received URL from IPC" and does not spawn a duplicate | **#8** mutex/GetLastError, **#11** IPC retry |

In the UI tester (0.1), also confirm: opening `--window` does **not** pop the Windows Settings app (validates **#9/#10** — the default-browser prompt bug).

## Tier 1 — Default-browser integration (needs registration)

Registration options:
- **Bundled `.reg` (no Inno Setup needed):** double-click `finicky-register.reg` (assumes the exe is at `C:\Finicky\`; edit the path inside if you placed it elsewhere). Clean up afterward with `finicky-unregister.reg`.
- **Or the installer:** build `scripts/installer.iss` with Inno Setup (free) → run the resulting `FinickySetup-*.exe`.

Then:
1. Settings → Default apps → set **Finicky** as the browser (ProgID `FinickyURL`).
2. Click an `http(s)` link from another app (email, a saved `.html`, a chat client).
3. **Pass:** the link routes through Finicky to the correct browser, and **Settings does not reopen on every click.**
4. Startup logs should read **"Finicky is the default browser"** once set (validates **#9** ProgID match).

## Tier 2 — Depth

- **Profiles:** a rule targeting a specific Chrome/Firefox profile opens that profile (`--profile-directory=` / `-P`).
- **keepRunning:** with `keepRunning` set, the process stays resident (Task Manager) and routes multiple links without relaunching.
- **Autostart (if installer used):** after reboot, Finicky is resident but **no config window pops** (validates the installer autostart `--window` fix).

---

## Capture for any failure

- Exact action (what was clicked / command run) and what happened vs expected.
- Logs: `%APPDATA%\Finicky\Logs`
- Screenshot of any blank/frozen window.

Fix on branch `windows-support` (`~/Projects/finicky`, remote `fork` = `chaoz23/finicky`), then re-run the affected tier. If all green, post results to PR #542 and mark it ready for review.
