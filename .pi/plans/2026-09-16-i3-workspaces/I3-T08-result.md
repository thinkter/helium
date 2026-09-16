# I3-T08 — geometry-backed resize mutations and native divider requests

2026-09-16. **Implementation complete; runtime acceptance deferred, NOT passed.**
Only T08 implemented. No tests/browser processes were executed, no resources
restored/downloaded, and no T09 controller or production routing added. The
local untracked plan has an implementation-only checkbox/note; it remains
unstaged. Validation follows tracked `validation-status.md`.

## Delivered

- `geometry.patch`: exposes each visible split's realized tree-order pixel sizes
  and recursive minima from the existing allocation pass. No duplicate minimum
  algorithm, model-file change or Views dependency in the policy layer.
- `ResolveWorkspaceResizeTarget(model, axis)` walks structural selection and
  ancestors, picking the nearest matching non-unary split. A selected container
  uses its selected child; an ancestor uses the branch containing selection.
  Prefer that child's following divider, falling back to its preceding divider
  for the last child. Hidden selection has no target; tabbed axes are rejected.
  Viewport/constrained feasibility is checked at mutation time, not resolution.
- `ResizeWorkspace(model, width, height, metrics, target, delta_pixels)` is the
  common pointer/keyboard mutation boundary, recomputing authoritative geometry
  and rejecting invalid, hidden, unary, tabbed or constrained targets. Positive
  delta grows the preceding **tree-order** child, including in RTL. It clamps
  transfer against both adjacent subtrees' recursive minima, including selected
  tabbed header costs, and commits a validated candidate model.
- The resized container's realized pixel allocation is materialized into
  normalized weights, then only the adjacent pixel allocation changes. This
  deliberately replaces that container's latent minimum-clamped weight ratios
  with what is actually displayed: otherwise dragging one divider can move
  unrelated siblings. Nonadjacent current pixel extents, other containers'
  weights, topology, selection and active page are retained. A zero/clamped-zero
  request leaves even the original weight representation unchanged.
- `views.patch`: adds `WorkspaceResizeArea`, a native contents-free ResizeArea
  and its own delegate, NOT MultiContentsView's pair delegate. Native cursor,
  mouse/touch drag handling, focus ring, axis arrow keys and AX increment/decrement
  feed a single ID/delta callback. Pointer deltas are cumulative, keyboard/AX
  deltas are single steps. Explicit geometry RTL and DIP-to-pixel rounding are
  applied once; keyboard step is 50 DIPs (not a registered browser shortcut).
- Slider AX role, axis, recursive min/max/current pixel range, value text and
  owner-requested committed-value announcements. No optimistic value/weight
  updates in the surface. English harness labels need localization before
  exposure. Invalid Update preserves the existing surface; Update during an
  in-progress drag is rejected.
- Host renders one **display-only** divider per geometry divider, with shared
  physical-edge-to-DIP conversion. Tabbed headers never become resize surfaces;
  constrained projection has no dividers. Clear removes dividers. Every host
  divider has a null callback and is disabled, including direct AX actions.
  `SetCanProcessEventsWithinSubtree(false)` and the default-disabled feature
  remain unchanged; there is still no production construction path.

## Actual native audit / input contract

Read the actual ResizeArea, MultiContentsResizeArea, Slider accessibility,
WorkspaceContentsView, model and geometry implementations, plus the plan,
scout, workflow, T05/T06/T07 results, validation policy and CONTRIBUTING.

`views::ResizeArea` reports cumulative **screen displacement in DIPs**, not
incremental deltas. It flips horizontal reports with global UI RTL. The new
adapter undoes that convention when it differs from explicit geometry RTL;
keyboard arrows use geometry RTL directly. Unlike native pair resizing, it
never swaps pages on double-click/tap, uses native split ratios, or writes
SessionService metadata. Gesture handling remains the native primitive's path.

The base capture-loss handler feeds its stored screen coordinate through a
local-to-screen conversion. The adapter lets the base clear its private drag
state but discards that synthetic displacement, finishing with the last
requested cumulative displacement. A finished drag does not emit a second
capture-loss completion. Keyboard/AX resizing is rejected during a drag.

Callbacks are copied before dispatch; there is no member access after Run,
allowing synchronous owner destruction. No WebContents or model pointer is
retained. A presented ID is not authority over a newer tree.

### Precise T09/T10/T13 handoff

1. Bind callbacks with weak controller lifetime and workspace/presentation
   generation. Revalidate current IDs, lifecycle state, viewport and metrics.
   Reject stale requests even when a divider index still exists after reorder.
2. On the first cumulative pointer callback, snapshot the authoritative model,
   target, viewport and metrics. Each cumulative request must call
   `ResizeWorkspace` on that **same baseline**, not repeatedly add its delta to
   the previously mutated state. Commit through the controller's transaction
   boundary. On `last_update`, finish/discard the baseline. Capture loss finishes
   at the last requested displacement; it is not rollback semantics.
3. Freeze the surface identity/metrics for the gesture. Keep the captured surface
   alive while reconciling page geometry; update only its committed AX value
   with `UpdateValue`. Rebuild surfaces after completion. Cancel/invalidate on
   external topology, workspace, scale, viewport or lifecycle changes; do not
   rebase a cumulative drag silently. Do not reuse the harness's recreate-all
   Present strategy as a production drag reconciliation algorithm.
4. Keyboard commands resolve with `ResolveWorkspaceResizeTarget`; focused
   divider keys/AX already provide a target. Both enter `ResizeWorkspace`, with
   positive delta defined as preceding tree-child growth. Commands that mean
   grow the *selected* last child must deliberately invert delta. T13 owns the
   final command semantics/default accelerators, not this surface.
5. Reconcile actual browser activation/visibility and announce **committed**
   values only. No native pair/session metadata writes. Do not lift host event
   suppression or attach real browser tabs before the T06/T09/T10 contracts are
   implemented and runtime-validated. Standalone contents-free surfaces are the
   only input-capable boundary in this slice.

## Compilation/static verification (not tests)

Starting C HEAD `b8999d0422d295d9d5481076076c968256ad792e`.
T HEAD remains `fbd20c49f1c3a8d0057a8714e5c9e98539cd2a21`; P HEAD remains
`2686b7c1cff1daccb5d5fbfa15c63a00fb236814`. C/P/T/S definitions and container
quilt wrapper are in `implementation-workflow.md`. Evidence is ephemeral under
`/tmp/i3-t08/`.

- Preflight merged/canonical replay: **348 patches / 1,637 files**, fuzz=0,
  zero differences. C/T series and merged/applied order matched. Saved statuses,
  patches/.pc archive, existing workspace sources and UI BUILD before edits.
- Final production compilation: **9 steps succeeded**, GN generated 32,685
  targets / 4,981 files. Compiled all three changed production objects using
  actual Chromium flags on the exported, platform-reapplied source:

  ```bash
  docker run --rm --network none --read-only --tmpfs /tmp \
    --user "$(id -u):$(id -g)" -e HOME=/tmp -e SCCACHE_DISABLE=1 \
    -v "$P:/repo:rw" -w /repo/build/src --entrypoint bash \
    chromium-builder:trixie-slim -c '
      buildtools/linux64/gn gen out/Default &&
      third_party/siso/cipd/siso ninja -C out/Default -local_jobs 2 \
        obj/chrome/browser/ui/ui/workspace_layout_geometry.o \
        obj/chrome/browser/ui/ui/workspace_contents_view.o \
        obj/chrome/browser/ui/ui/workspace_resize_area.o'
  ```

- **Both test objects compiled, not linked/run**, final exit 0, no diagnostics:
  `workspace_layout_geometry_unittest.o` (736,024 bytes) and
  `workspace_contents_view_browsertest.o` (640,216 bytes). Used `siso query
  commands -C out/Default` for the exact unit_tests/browser_tests objects;
  selected only each source's compiler command, retaining flags/includes/modules/
  sysroot and replacing only its object/depfile output prefix with `/e/<name>`.
  Scripts `compile-<name>.sh`, query `commands.log`, final `final-build.log`.
  Executed in `/repo/build/src/out/Default` with `/tmp/i3-t08:/e`. This does not
  build either scheduled test target or bypass inputdeps for a runnable binary.
  The known missing fixture remains untouched; no full test build was attempted.
- Four new `WorkspaceResizeTest` cases specify nearest axis/selected container/
  last-child targeting, hidden selection, recursive header minima, unary
  ancestor skipping, adjacent transfer, fractional DPI/RTL equivalent weights,
  extreme negative clamp and invalid/constrained no-ops. Native browser-test
  source adds contents-free pointer/key/AX request equivalence, capture loss,
  committed AX values and disabled host-divider checks. **All NOT RUN.**
- Final and post-build merged/canonical replay: **348 patches / 1,639 files**,
  fuzz=0, zero differences. Owned source hashes match after narrow pop/push,
  suffix reapplication and final compilation. `git diff --check` passed.

## Patch layering / preservation

Kept canonical C authoritative. Popped suffix **and views.patch** to geometry;
refreshed geometry only, verified empty quilt delta, popped/pushed it with exact
owned hashes, then reapplied views without force/refresh. Its existing GN hunks
applied unchanged. Registered new view files before creation, edited S and
refreshed views only, verified its empty delta and owned pop/push hashes, then
reapplied all nine platform patches. Repeated the narrow sequence for a test
review addition. No model adapter in views.patch and no model.patch change.

Inspected all generated patch differences, including final test/comment/include
changes. Scratch unmerge `/tmp/i3-t08-export-ypmgg57w` verified unchanged generic
and original platform series. Exported only geometry.patch and views.patch
E -> T -> C. P/T/C SHA256:

```text
8e5231ddb69a0ba3b09ab117454529e5a4fb8387a210cb131bae09abe074207e geometry.patch
b51f9acb950bcb90a47801b17850cb628ee93a572d2b344d7f4a444e920f5115 views.patch
```

Of **696** preexisting live patch/metadata files, only those two changed.
All **339** C/T generic patch pairs match; no series changes. Model remains
`e341a2d356c51696734da2c90d16e1c7508902d6a898e759941f4b67acd26185`.
P/T status path lists match preflight; tooling remains intentionally dirty.
Deleted AGENTS, dangling CLAUDE, build-local.log and untracked scout/plan/blocker
were not restored/staged/modified by source tooling. A preservation-script
assumption that build-local.log was empty (from prior result prose) failed:
it contains an existing missing docker-build.sh diagnostic, dated before this
task. It was left untouched; no cleanup was performed. No setup/reset/pull,
broad sync, unrelated patch refresh or resource restoration. No commit skill
was available; used CONTRIBUTING's scoped message and explicit task paths.

## Acceptance limits / ISC

ISC-2/3/4 resize foundations are implemented and compiled: recursive minima,
non-tabbed dividers, topology-preserving normalized-weight mutation. Their
runtime acceptance is **deferred**, not checked as passed. Pointer/touch capture,
AX announcements, focus/paint/hit-target usability, RTL interaction, repeated
live drag reconciliation and ordinary native-split regression all need later
explicitly authorized runtime validation. T06/T07 acceptance remains deferred.
No foreground/activation/lifecycle safety claim or feature readiness follows
from these objects. Existing T06 primary AX/dialog/DevTools/fullscreen caveats
remain. No controller, command/settings integration, persistence, or exposure
was implemented.
