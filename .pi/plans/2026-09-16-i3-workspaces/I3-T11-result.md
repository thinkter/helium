# I3-T11 — lifecycle and native split exclusion

2026-09-17, sequentially after T10 closure `d32c91a` (initial `25844e9`).
**T11 IMPLEMENTATION COMPLETE; runtime/ISC acceptance UNVERIFIED.**
Both experimental gates remain OFF by default. No public enable commands.

## Finite implementation criteria and contracts

1. **Authoritative exclusion before native membership.** TabStripModel now owns
   a Views-independent native-split transition policy. Workspace tracking installs
   it before native transfer and removes it only after reverse transfer. Native
   UI can query `IsNativeSplitAllowed()`, without relying on an observer detecting
   an already-created split. `AddToNewSplit` and `RestoreSplit` invoke the policy
   before mutation; the sole `CreateSplit` implementation asserts native mode.
   Detached collection insertion scans for split membership before admitting the
   collection, covering both direct split transfers and groups containing splits.
2. **Data-preserving low-level transition, not rejection with a fake ID.**
   The native creation API returns a real non-optional SplitTabId; restore and
   transfer carry authoritative session/owned collection data. A low-level call
   synchronously stops workspace tracking and completes ordinary-host restoration
   FIRST, then proceeds with the existing native operation. It never silently
   drops/dissolves the incoming split or fabricates a successful ID. There is no
   state with native memberships inside active tiling. This deliberately yields
   the experimental layout, not the existing native split/session data. Re-enable
   still refuses windows with native splits. No retained rollback project added.
3. **UI creation disabled while tracking.** New-split command enablement checks
   model policy, including dynamic execution-time queries. Start/stop refresh
   command state. `chrome::NewSplitTab` returns before creating a blank tab;
   split context menu enablement and `ExecuteAddToNewSplitCommand` reject the
   action. MultiContentsView's native drag/drop enablement uses the same model
   policy. A stale/unusual low-level native caller still hits criterion 1 rather
   than bypassing exclusion. Ordinary windows have no installed policy, retain
   native command/drop eligibility and run their original native mutations.
4. **Confirmed removal only.** Existing WillDetach continues to revoke queued
   input, not attachments or tree nodes. Confirmed pre-removal detaches the native
   page, and kRemoved repairs the model and retires the handle/subscriptions.
   Canceled beforeunload is structurally a no-op; callback epochs are refreshed
   even when cancellation produces no subsequent strip change.
5. **Metadata/transfer/final-tab lifecycle.** Pin/group/reorder operations stay
   Chromium metadata, not tree containers or tree-order commands. Stable handles
   and kInserted/kRemoved reconciliation preserve window ownership; replacement
   retains its association. Added explicit final-tab handling: after confirmed
   removal empties the strip, stop tracking instead of presenting an empty native
   tree or creating a replacement tab. Chromium alone owns close/tear-out/window
   lifecycle. Existing normal cross-window and incognito regression source is
   retained; native collection transfers now get the pre-insertion guard above.

These are finite source/compilation criteria, NOT claims that browser scenarios
or ISC-7/11/A-1/A-2/A-4 have passed runtime acceptance.

## Compile-only regression source (NOT RUN)

In `workspace_controller_browsertest.cc`:

- `NativeSplitPolicyYieldsBeforeRestoreAndTransfer`: disabled command/context
  menu/drop queries; new-split action adds no tab; session restore retains the
  supplied split ID while leaving tiling; re-enable refuses the split; transfer
  of a group containing a split preserves group/split identity; direct split
  collection transfer preserves the same split ID at another tracking window.
- `StripMetadataPreservesTreeAndFinalDetachRetiresPolicy`: strip reorder,
  pin/unpin, group/ungroup preserve layout child order and tab association;
  confirmed removal validates the tree; final detach clears tracking/native host
  and restores native-mode eligibility without manufacturing another tab.
- Extended `CanceledBeforeUnloadKeepsCommittedAttachment`: after cancellation's
  existing attachment/active/foreground assertions, request close again, ACCEPT
  the beforeunload dialog, wait for contents destruction, assert association
  retirement, remaining tab count and valid model. This tests cancellation and
  actual confirmation separately in source; neither scenario was executed.
- Existing `UnsupportedWindowsAndNativeSplits` compiles ordinary native creation,
  enable refusal and direct low-level creation yielding tracking. Existing
  `DetachTransferAndIndependentIncognito` compiles stable-handle normal transfer,
  independent destination/private controllers and real window teardown.

No new ownership layer, persistent tab indices or session serialization added.

## Compilation evidence

`/tmp/i3-t11/final-compile/results.log`: **12 exit codes 0**, all compiler
logs empty. Includes the seven directly relevant production/test objects and
five dependent integration objects:

```
browser_command_controller 0
browser_commands 0
browser_view 0
browser_window_features 0
contents_border_controller 0
multi_contents_view 0
tab_strip_model 0
workspace_contents_view 0
workspace_controller 0
workspace_controller_browsertest 0
workspace_resize_area 0
workspace_tabbed_header_view 0
```

Existing nine per-source commands are byte-identical to the proven
`/tmp/t10-pane-input/final-compile` commands. The three newly affected production
sources use their OWN generated commands from the existing Ninja graph:
`obj/chrome/browser/ui/ui/{browser_command_controller,browser_commands,multi_contents_view}.o`.
Saved original generated command lines under `compile/*.generated`; removed only
sccache and redirected `-o`/`-MF` to evidence. All flags/module/sysroot settings
preserved. Used existing `/tmp/ninja-bin/ninja -t commands`, no dependency build
or graph changes. Compilation used existing `chromium-builder:trixie-slim`,
network disabled, source read-only at `/repo`, outputs `/evidence`. Final source
was fully patched, including platform suffix. No linking or test execution.

## Source/patch provenance and preservation

- T10 closure final FULL replay was T11's immediately preceding clean baseline:
  canonical and merged **349 patches / 1643 paths / zero differences**.
- `/tmp/i3-t11/quilt-patches.tgz` snapshots patches and complete quilt metadata;
  `sources/` saves all 22 registered paths before source edits. Verified owned
  controller.patch top. Registered browser_commands.cc,
  browser_command_controller.cc and multi_contents_view.cc BEFORE editing.
  All three were previously patched paths covered by full baseline replay;
  their new quilt originals match pre-edit snapshots byte-for-byte. No formerly
  unpatched source was touched, so no upstream baseline was invented.
- Narrow refresh of controller.patch only; owned pop/push reproduced **22/22
  registered source hashes**. Normal platform suffix reapplication, no force.
  Empty quilt delta; applied order matches merged series; final top remains
  `helium/linux/disable-tab-strokes.patch`.
- Scratch unmerge verified generic series against canonical/transport and
  platform series against series.orig. Only controller.patch copied through
  scratch -> transport -> canonical. Final SHA256 identical at all three:
  `49863198fa1cab98fedbf2c16f748e820743ad4b66ceb3c39c3f16c85044b79f`.
- Final FULL independent canonical AND merged replay: **349 / 1643 / zero
  differences**, `/tmp/i3-t11/final-audit/`, canonical platform input from P HEAD.
- Archive comparison: only controller.patch changed; only the three registered
  controller originals were added to .pc. Existing repaired metadata and all
  non-owned patch bytes unchanged. Platform/transport status listings match
  pre-T10 listings; intentional owned transport patch content is updated.
- `git diff --check` flags ten single-space unified-diff context lines and the
  final context blank line in quilt output, not added C++ whitespace. Deleted
  AGENTS.md, dangling CLAUDE.md, untracked plans/build-local.log and unrelated
  dirty platform/transport state preserved. Explicit task-path staging only.

## Handoff and remaining work

**T11 implementation is done: main agent should send the requested PING.** This
is not a ready/runnable browser handoff. Human manual tests are requested once a
runnable, authorized validation build exists; they were NOT performed here.
T12 still owns modal positioning/accessories, fullscreen and DevTools policy;
T13 owns public commands/settings exposure. T14/runtime validation must exercise
canceled/accepted close, tear-out, groups/pins/reorder, native split/session
restore and ordinary-window regressions, plus the T10 existing-dialog attachment
case. No feature default was enabled and no ISC acceptance checkbox is passed.
No browser launch, tests, resource restoration/downloads, setup/reset/pull or
broad synchronization occurred.
