# T10 post-repair compilation and ownership checkpoint

2026-09-17. **T10 remains incomplete; T11 has not started.** No runtime or ISC
acceptance is claimed. Live model environment was `openai-codex/gpt-6-astra`.

## Starting-state reconciliation

Canonical HEAD was `ee7a75f`. The interrupted worker left the platform suffix
popped: 340 applied entries, `controller.patch` on top, not 349 fully applied.
`quilt diff -z` found four unrefreshed files:

- `workspace_controller.cc`: include cleanup and an unsafe access to `window_`
  after synchronous activation (despite its own no-access-after-callback rule).
- `workspace_controller.h`: declarations for display resolution, header/resize
  callbacks and a resize transaction, with **no corresponding implementation**.
- `tab_strip_model.{cc,h}`: a public pending-projection clear function used by
  that unsafe post-activation call.

Before edits, all patches and `.pc` were archived, the four partial sources
were copied, and both full applied-prefix replays were performed. Each applied
340 patches to 1619 source paths with exactly those four differences, no other
source differences and no patch failures. The incomplete declarations are
preserved in the snapshot, not shipped as fictitious implemented contracts.

Evidence root: `/tmp/t10-astra-ndkz/` (ephemeral). Its `quilt-patches.tgz`, copied
sources, initial replay trees, `initial-compile/`, generated commands, and final
verification artifacts remain available. No manual `.pc` changes were made.

## Changes shipped

- Fixed the four concrete BrowserView compiler errors: iteration over native
  `raw_ptr` containers, assigning a Views-owned child to `unique_ptr`, and
  converting displayed plain-pointer containers to the declared raw-pointer
  vector. Workspace host ownership now follows the existing Views child-tree
  pattern; removal and BrowserView destruction clear the borrowed pointer.
- Revoke primary AX designation before native detachment/replacement and on
  host clear. Restore the ordinary native primary when reversing the transfer.
  This does not establish the remaining generation/active-identity guarantees.
- Foreground queries now read only the committed provider, never the pending
  selection vector. Transition notification snapshots the old committed set
  and consumes pending intent **before** callbacks. No-op selection also clears
  intent inside TabStripModel before observer dispatch. Empty pending input is
  treated as an abort. Removed the interrupted worker's unsafe controller
  access after `ActivateTab()` returns.
- Reject a displayed-key publication if any key no longer resolves, rather
  than calculating and discarding a validated vector and publishing anyway.
- Updated obsolete observation-only/no-primary comments to describe the
  existing internal transfer path without claiming complete integration.

Tabs/WebContents remain externally owned; stable tab handles and keys remain
unchanged. Both experimental features remain disabled by default. No public
entrypoint, shortcut, flag UI, or command was added.

## Compilation evidence

First compiled before feature edits using the generated per-object commands
from `out/Default` subninjas in `chromium-builder:trixie-slim`, with P mounted at
`/repo`. The bundled `third_party/llvm-build/Release+Asserts/bin/clang++` exists
and executed successfully. BrowserView initially failed with the four errors
above; the other seven objects compiled.

After export and final platform reapplication, all eight production objects
were compiled again with exit status **0**:

```
obj/chrome/browser/ui/ui/workspace_controller.o
obj/chrome/browser/ui/ui/workspace_contents_view.o
obj/chrome/browser/ui/ui/workspace_resize_area.o
obj/chrome/browser/ui/ui/workspace_tabbed_header_view.o
obj/chrome/browser/ui/ui/browser_view.o
obj/chrome/browser/ui/ui/contents_border_controller.o
obj/chrome/browser/ui/tabs/tab_strip_impl/tab_strip_model.o
obj/chrome/browser/ui/browser_window/internal/internal/browser_window_features.o
```

Commands were extracted by `ninja -t commands <object>` (graph inspection, not
execution), selecting the actual per-source compiler command. Only `sccache`
was removed and `-o`/`-MF` destinations redirected to `/evidence`; every compiler
flag, toolchain, generated header path and sysroot was preserved. Source mount
was read-only, networking disabled. Final command files, depfiles, objects and
empty diagnostic logs are in `final-compile/`; `result.log` records eight zeros.
No GN regeneration, graph edits, stubs, resource restoration, downloads,
automated tests, browser process or linked browser build were performed.

## Quilt/export evidence

- All edits were source-first at `controller.patch` top, with registered files
  and source snapshots before modification. Only that patch was refreshed.
- Owned pop/push reproduced **all 18 registered source hashes** (17 actual
  diff files; the inherited header registration has no diff).
- Platform suffix reapplied without refresh/force. Final top is
  `helium/linux/disable-tab-strokes.patch`; 349 applied entries equal
  `series.merged`; `quilt diff -z` is empty.
- Scratch unmerge generic/platform series matched canonical/`series.orig`
  byte-for-byte. Only the reviewed controller patch was copied through T to C.
- Final independent merged and canonical replays: **349 patches, 1643 source
  paths, zero failures, zero differences** in each mode. Canonical replay used
  original platform patches from P HEAD, not the live refreshed copies.
- The four repaired controller backups still have exactly the SHA256 values
  recorded in `T10-provenance-repair-result.md` (`4c955158…`, `84233e8b…`,
  `3334a1d8…`, `7262030b…`). Provenance repair has not regressed.
- Snapshot comparison found exactly one changed pre-existing patch-tree file:
  `helium/ui/workspaces/controller.patch`. Non-owned patches stayed unchanged.
- P/T/C controller SHA256:
  `a4b07b408fadeae8cbdeb7807d968fb114b194802f652f6c6e81da96f07b63eb`.
- `git diff --check` reports four newly exposed patch-context blank lines
  (single-space unified-diff context), not source whitespace errors. Quilt
  generated these; they were not manually edited out of the patch.

Deleted `AGENTS.md`, dangling `CLAUDE.md`, `build-local.log`, untracked planning
artifacts and dirty transport tooling were preserved. No commit skill exists
in the available skill locations; CONTRIBUTING's scope-first explicit-path
commit workflow was used.

## Exact remaining T10 obligations (source review, not result-prose claims)

1. **Atomic displayed transition is still incomplete.** Pending intent is now
   isolated correctly, but the controller still supplies the old displayed
   set before activation, not the destination geometry. External selection and
   background insertion need the same old/new committed notification boundary.
   `OnTabStripModelChanged()` still optimistically rewrites singleton
   `displayed_keys_` before `PresentHost()` succeeds. Remove that shortcut;
   compute notifications only for leaves actually leaving the committed set.
2. **Finish routing, not just getters.** Basic transfer/getters/layout compile,
   but `UpdateDevTools`, content-focused/stored-focus mapping, accessible panes,
   modal host iteration and other BrowserView methods still directly use
   `multi_contents_view_`. Active getter fallbacks can return the detached native
   slot during an uncommitted workspace selection. The early workspace branch
   in `OnActiveTabChanged()` omits normal accessories/focus work. Audit these
   before treating single-host transfer as complete.
3. **Presentation/start/stop lifetime and generation guards.** Provider binding
   still uses `Unretained`; native attach/detach and border callbacks can reenter
   presentation or teardown. Current `presenting_` protects recursion, not
   deletion or stop/restart publication. Check generation, current host, tab
   membership/contents and tracking after callback-capable operations. Empty
   bounds/startup/failed presentation need explicit rollback and foreground
   semantics; actual display scale/RTL metrics must be passed by the controller.
4. **Canceled unload and confirmed detach.** `WillDetach()` currently only
   increments generation, preserving the leaf and committed provider on a
   canceled close. Keep this property: it precedes confirmation. Complete
   confirmed removal/transfer attachment and notification ordering, plus
   cancellation-safe input/gesture invalidation, without preemptive removal.
5. **Scrims and primary AX.** Refresh scrims after successful new host/projection
   commits, not just existing tab-model notifications. AX demotion order is
   fixed, but exactly-one-primary must be validated against actual active tab,
   contents and presentation generation, including failed/stale presentations.
6. **Guarded renderer/footer/overlay focus.** Host event processing remains
   disabled, headers and dividers have null callbacks, and no renderer-focus
   adapter validates displayed key/contents/generation before deferred Chromium
   activation. Native focus is not isolated by disabling Views subtree events.
7. **Controller mutations and drag baseline.** Implement header selection and
   structural/resize reconciliation. Pointer cumulative deltas must use a frozen
   gesture baseline, keyboard/AX deltas must be incremental, and external
   generations must cancel obsolete gestures. `Present()` currently refuses all
   reconciliation while any divider is resizing. No gesture implementation was
   fabricated from the interrupted worker's declarations.
8. Existing controller browser-test source is stale: it enables only tracking,
   whereas `StartTracking()` now also requires the host feature, and its
   pre-detach assertion contradicts cancellation-safe resolution. Tests were
   neither changed nor executed in this checkpoint; reconcile their contracts
   before any future authorized runtime-validation phase.

T11 native-split model-boundary exclusion and T12 fullscreen/dialog/DevTools
completion remain prerequisites to T13/public exposure. No ISC checkbox was
marked passed; compilation and replay provide build/provenance evidence only.

## Human validation handoff

**Not runnable as a user feature yet:** no linked build or public enablement
path is available from this checkpoint. Do not present a flag-only launch as a
working tiling mode. Once T10–T13 provide a runnable build and documented entry:

1. In a disposable profile, open three distinguishable pages; split to show
   all three, click/type in each, and confirm toolbar, navigation and shortcuts
   target the clicked page while every displayed page remains foreground.
2. Switch a container to tabbed with a nested split child, switch workspaces,
   then select a hidden global tab. Confirm correct ancestor reveal, preserved
   tabs/topology, and no hidden leaf reported as displayed.
3. Cancel a beforeunload close; confirm unchanged page/leaf/layout/focus. Then
   confirm close and move tabs between windows; check no duplicate attachments.
4. Resize by pointer, keyboard and accessibility action; interrupt a drag with
   tab change/close, shrink below minima, restore size and change display scale.
5. Check screen-reader primary/focus traversal, modal scrims, renderer/footer
   clicks, DevTools, fullscreen exit and ordinary-window/native-split behavior.

These are future human checks, not tests run or results accepted here. The
requested T11-complete notification has not been sent because T11 is unfinished.
