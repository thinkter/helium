# I3-T10 transfer result

2026-09-16 — implementation continuation, runtime unverified.

## Defects fixed

- `WorkspaceController::SetDisplayedTabs()` now commits the displayed projection without leaving `TabStripModel`'s pending-selection override engaged. Pending displayed tabs are used only immediately before deferred Chromium activation, so later provider reads and fallback foreground reads cannot be permanently shadowed.
- `WillDetach()` is staged: it invalidates in-flight requests but does not make a still-attached page disappear from the committed projection. The confirmed remove observer remains the model/host publication boundary.
- BrowserView now performs an explicit transfer before workspace attachment: every native `MultiContentsView` page is detached and demoted, hidden, and only then is the workspace child exposed. Stop reverses this order by clearing the workspace first, then restoring the active native page. Active/per-page/all-visible container getters, active WebView lookup, tab-detach cleanup, and active-tab change routing use the committed workspace host while transfer is active.
- Workspace host bounds are refreshed from BrowserView's current content region on every layout pass and `PresentHost()` rejects empty geometry before publishing displayed keys. Scale/layout changes therefore re-run the retained host reconciliation instead of relying on startup sibling bounds.
- Workspace presentation demotes all retained leaves and designates exactly the model's active displayed leaf as the primary WebContents for accessibility; no hidden or inactive leaf is designated primary.
- Selection/lifecycle observer changes re-present the host after the model mutation, preserving detach-before-attach and retained leaf identity.

The feature flags and production entrypoint remain disabled; no user-facing exposure was added. Scrim/accessory projection, renderer-focus adapters, fullscreen/dialog/DevTools routing, drag transactions, and native-split exclusion remain intentionally outside this continuation and block exposure.

## Verification and provenance

- Source was edited in the applied Chromium tree and only `helium/ui/workspaces/controller.patch` was refreshed.
- An accidental initial refresh was corrected safely: platform suffix was popped through the owned controller patch, source snapshots were restored, controller was refreshed narrowly, and the complete suffix was pushed again. No platform patch was retained with task changes.
- Scratch unmerge routed only the owned controller patch to generic output. P/T/C controller patch hashes match:
  `94cda3530cfdad65410b33f9ad599ec08239c0e67a959f024adf4bba16e735dc`.
- Applied quilt state ended with the complete stack pushed and `quilt diff -z` empty.
- Production compilation was attempted in the verified `chromium-builder:trixie-slim` container with the exact generated siso target command. The graph reached 941 steps but stopped at an unrelated generated Dawn source action because Go attempted to download `go1.25.0` and network is disabled. No source compiler was reached. Exact per-source generated clang commands were queried; this container mapping does not contain the referenced Chromium LLVM toolchain, so an honest source-object fallback could not execute. No tests, browser process, resource restoration, or downloads were run.
- A post-edit merged replay was attempted with the canonical reconstruction procedure. It stopped at `controller.patch`: the live `.pc` backup footprint contains already-applied prior controller hunks in `tab_strip_model.cc`/BrowserView, so the replay reported reverse/failed hunks rather than silently accepting fuzz. This is recorded as a provenance blocker, not a pass; no runtime or test acceptance is claimed here.
