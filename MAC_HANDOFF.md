# Handoff: Windows box → Mac session (2026-07-03)

You are the Mac-side Claude session coordinating the Finicky Windows port (fork
`chaoz23/finicky`, upstream draft PR johnste/finicky#542). The Windows instance
has finished everything it can do on-box. This file is your work order; the
evidence behind every claim is in `SMOKE_TEST_FINDINGS.md`.

## State of the world

- **Branch `windows-smoke`** (this branch) holds everything: code fixes +
  testing docs. Key commits since your `d62f9c4`:
  - `7babaf3` — **code**: UI↔Go bridge fix (F3 root cause), named-pipe IPC
    (F8), Windows version read via PE resource (F7 Windows half), platform-
    agnostic tests. This is the one commit that must be promoted.
  - `f1c2577`, `aa61d03`, + this one — **docs only** (findings/verification).
- **Windows box status:** build `7babaf3` verified end-to-end on Win11 Home —
  Tier 0 complete, Tier 1 complete except the set-default-browser human step
  (registration, OS `finicky://` handoff, and IPC through the *installed* exe
  all pass), Tier 2 profile routing passes, installer full lifecycle
  (compile → silent install → uninstall → reinstall) passes with no UAC.
  GUI verified by owner screenshots: dropdowns paint Chrome/IE/Edge (F3),
  multi-field rules persist (F4), no Safari (F1), About wording (F6).
- **Corrections to your prior notes:** F3 was NOT `detect_windows.go` —
  detection worked; the whole UI→Go bridge was dead (App.svelte webkit-only
  `sendMessage`). Your F7 version read: confirmed, and its Windows `defaults`
  half is fixed; the stale-4.2.2 half is yours (below).

## Your tasks, in order

1. **Ratify the `github.com/Microsoft/go-winio v0.6.2` dependency** (direct).
   Grounds: Go's AF_UNIX on Windows fails intermittently with WSAEINVAL after
   socket create/remove cycles (reproduced deterministically on-box; see F8) —
   the URL listener was silently dead. go-winio is the containerd/Docker named-
   pipe library and drops in via the same net.Listener/net.Conn interfaces.
   If vetoed, the alternative is hand-rolled overlapped-I/O named pipes; there
   is no viable AF_UNIX path.
2. **Cherry-pick `7babaf3` → `windows-support`** (code only; never the testing
   docs). Verify macOS build + tests after — the commit touches shared files
   (`App.svelte`, `types.ts`, `version.go` split, resolver/rules tests) but was
   written to keep macOS behavior identical (webkit bridge tried first; darwin
   version file unchanged; tests assert `browser.DefaultBrowserName`).
3. **F7 part 2 — real version number** (Mac side by prior agreement):
   `var version string` in `version.go` set via
   `-ldflags "-X finicky/version.version=$(git describe)"` in build-windows.sh
   / CI; make `version_windows.go` prefer the injected value and fall back to
   the PE resource; align `versioninfo.json` / `installer.iss` / winget
   manifest off hardcoded 4.2.2 (real base ≈ v4.4.0-alpha; scheme = johnste).
4. **Batched UI round:** F2 (Test-tab timeout + error state) and F1's last
   site (StartPage.svelte hardcoded Safari seed → seed from backend
   `browser.DefaultBrowserName`).
5. **Installer polish before release:** change `DefaultDirName` to
   `{userpf}\Finicky` — the current `{localappdata}\Finicky` collides with the
   app's cache dir (cache/bundles sit beside the exe and get wiped by
   `[UninstallDelete]`). Also move `.syso` generation into build-windows.sh/CI
   instead of the committed binary (existing TODO).
6. **PR #542:** once 1–4 land on `windows-support`, post the smoke results
   (`SMOKE_TEST_FINDINGS.md` is the evidence packet) and flip out of draft.
   F9 (glob-vs-hostname rule UX + the too-narrow wildcard warning) is a
   shared-UI product question — surface it to johnste, don't fix unilaterally.

## Release-path notes (beyond the PR)

- CI `build-windows` job: UI build → asset copy → `.syso` regen → go build
  with version ldflags → ISCC installer → artifact.
- Distribution: prioritize winget (`scripts/winget/` exists) over a code-signing
  cert for the first release; unsigned = SmartScreen warning, document it.
- Update check: `apiHost` is empty on Windows (warns every launch) — point at
  a Windows-aware release feed or suppress cleanly.
- **Accepted risk (owner decision 2026-07-03):** single-machine validation
  only (Win11 Home). No Win10/no-WebView2 test happened. If anything, test the
  installer's WebView2 story on Win10 before announcing broadly.

## Still open on the Windows box (needs the human)

Set Finicky default in Settings ▸ Default apps → click links from real apps →
confirm routing + "Finicky is the default browser" log + no Settings popup.
The box is staged: installed (autostart on), registered, resident router
running. Cleanup path: uninstaller removes everything; reset default browser
in Settings.
