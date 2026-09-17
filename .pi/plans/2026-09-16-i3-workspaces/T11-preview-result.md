# T11-only developer preview

2026-09-17, starting canonical `c83f1fe65d5c82df999d6108bb077565f80a3ebe`.
Implemented by Astra (`PI_MODEL=gpt-6-astra`). **Build in progress; NOT yet a
ready-to-run handoff. No browser or automated tests executed.**

## Limited implementation

New isolated `helium/ui/workspaces/preview.patch`, immediately after controller
and before the unchanged platform suffix. Both features remain default-off.
Requires `--helium-workspace-developer-preview`, `WorkspaceTabTracking` and
`WorkspaceContentsHost`, plus an absolute `--user-data-dir` basename starting
`helium-workspace-preview-`, an actually newly created profile pref store,
normal non-incognito window, one `about:blank` tab and no native split.
`Profile::IsNewProfile()` was deliberately NOT used: it can return true on
application first-run with existing preferences.

BrowserView's normal layout schedules a weak, non-nestable startup callback
only after tabs and nonzero bounds exist. Widget close/fullscreen/readiness is
rechecked, visibility retries are bounded (10 x 100 ms), and one process-wide
claim prevents repeats. Three ordinary background tabs and a warning-page
navigation are created only under that explicit opt-in. No test friend or
WebContents owner is introduced. The real StartTracking transfer runs first;
a validated candidate uses CreateWorkspace/SplitLeaf/SetLayout/RevealTab to
build `H(left, Tabbed(single, V(upper, lower)))` with the same real tab handles.
A single PresentHost commits actual attachments/foreground. Existing headers,
dividers, pane input and ordinary tab lifecycle provide the manual actions.

This is not T12/T13. Fullscreen requests/transitions and DevTools Show (including
Inspect Element, docked and undocked) deliberately yield preview tiling back to
the ordinary host; no automatic topology restoration. Ordinary controllers do
not enter these preview-only stops. The warning page and
`docs/t11-workspace-preview.md` explicitly exclude unvalidated dialogs,
permissions, find, side panels/accessories and broader runtime safety.

## Source and patch evidence

Evidence root `/tmp/t11-preview`:

- Preflight full canonical AND merged replay: **349 patches / 1643 paths / zero
  differences**, `preflight/results.log`. Current applied order verified.
- `quilt-patches.tgz` snapshots all patch/quilt metadata; `sources/` saves owned
  files before edits. All four source files registered before mutation.
  Previously unpatched `devtools_window.cc` compared byte-identical against
  the existing Chromium 153.0.8010.36-lite source archive before registration.
- Only preview.patch refreshed. Owned pop/push reproduced **4/4 SHA256 hashes**;
  empty quilt delta and normal suffix push; applied order equals merged series.
- Scratch unmerge `final-export/` verifies generic and original platform series.
  Only preview.patch exported scratch -> transport -> canonical, with the one
  new series entry. All three copies SHA256:
  `13781443ab40e649acf99e46315b967df0b40637789295f7036380d393022df9`.
- Final full canonical AND merged replay: **350 patches / 1644 paths / zero
  differences**, `hardened-audit/results.log`.
- `preservation.py`/`preservation.txt`: all pre-existing patch bytes unchanged
  except merged series. All existing quilt originals unchanged except the
  necessary downstream `disable-tab-strokes.patch` BrowserView backup; that
  backup independently equals its snapshot plus ONLY preview BrowserView hunks
  with fuzz=0. Repaired metadata and unrelated platform dirty state preserved.
  Canonical deleted AGENTS.md, untracked plans and build-local.log untouched.
- Quilt context-only blank lines trigger `git diff --check`; no added C++
  trailing whitespace. No commit skill is installed; CONTRIBUTING scope-first
  commit convention and explicit task-path staging are used.

## Build provenance

`final-compile/results.log`: workspace_controller, browser_view and
devtools_window production object compiles each exit **0**, empty diagnostics.
Existing generated commands retain all compiler/module/sysroot flags; only
sccache removal and scratch output/dependency destinations differ. New DevTools
command was extracted from its actual Ninja target. No test objects or tests
were run. An initial scratch attempt used the wrong working directory and
failed 127; corrected to `/repo/build/src/out/Default` before successful runs.

Real browser target: `third_party/siso/cipd/siso ninja -C out/Default -offline
-local_jobs=2 chrome` in verified image
`a8356d8b1656c03670648b9eacab1dd009e75c026bd89eff8d5ced5878085c44`, source
mounted `/repo`. Existing out/Default args retained. Initial resources: 79 GiB
available disk, 30 GiB available RAM (38 GiB total). Logs retained under
`build/`; no arbitrary build timeout, test-resource restoration, network,
installation or dependency-graph bypass.

First graph attempt failed at Dawn: required Go >=1.25, image has 1.24.4.
Inspection found Dawn's existing Go launcher symlink targets `/usr/bin/go`,
so merely putting host Go on PATH was insufficient. Existing host static Go
1.26.5 is mounted read-only both as GOROOT and at that executable path; module
cache is read-only, GOPROXY=off, GOTOOLCHAIN=local, Docker network disabled.
Dawn generation subsequently succeeded, with no toolchain download. Sccache is
disabled only as a cache wrapper; no compiler or production flags changed.

The cached-Go run was gracefully interrupted after 931 steps / zero failures
for the strict new-pref-store safety hardening, before any source pop/edit.
Final sources were recompiled, exported and replayed before restarting the
same full graph. After observing ~27 GiB available RAM, compiler parallelism was
bounded to four jobs on restart. Build/resource monitor keeps a 12 GiB disk
reserve. Active container: `t11-preview-build-final`; exact invocation saved
in `build/run.sh`, output `build/final-build.log`.

**Final link status and binary freshness: pending.** The pre-existing
`/home/ashman/Documents/rubbish/helium-linux/build/src/out/Default/helium`
(dated September 16, 03:19 local) is NOT claimed to contain this preview.
Pre-build hash saved in `build/pre-build.sha256`.

## Human handoff

Use only the exact disposable-profile launch and focused expected-outcome
checklist in `docs/t11-workspace-preview.md`, after full build success is
recorded. No ISC runtime acceptance, T12/T13 completion, production safety or
ready-to-run browser is claimed from object compilation.
