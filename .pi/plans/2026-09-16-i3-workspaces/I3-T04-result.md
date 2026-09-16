# I3-T04 — workspace, insertion and confirmed-removal mutations

2026-09-16. **Accepted; T05 ready.** Only T04 implemented. The local file-backed
T04 checkbox is complete; preexisting untracked plan/scout remain unstaged.
This tracked result records completion without adopting unrelated files.

## Delivered / API contract

Refreshed only `patches/helium/ui/workspaces/model.patch`, editing its existing
three source/test files in S. GN registrations remain unchanged. No geometry,
controller, browser ownership, feature exposure, or persistence was added.
T03 structural navigation/layout/explicit-child activation semantics remain.

- `CreateWorkspace(initial_tab, workspace_ids, node_ids)` returns its new ID
  and activates it. The tab is an already-created opaque association; null
  creates a shell. Duplicate associations/invalid state return null unchanged.
  Existing workspace/node ID collisions are skipped. Generators stay outside
  snapshots and must not recycle removed IDs.
- `SwitchWorkspace(id)` restores that workspace's stored active page AND its
  independent structural selection by changing only active_workspace. It
  retains all trees and never creates a tab when the destination is empty.
- `SplitLeaf(leaf, horizontal_or_vertical, new_tab, node_ids)` replaces exactly
  the specified leaf's slot with a two-child split, preserving the parent's
  weight. Supports singleton roots and hidden/inactive targets. Reveals and
  activates the new leaf in its workspace; structurally selects the new leaf.
- `InsertTab(tab, opener, disposition, node_ids)` inserts immediately after a
  known opener, in that opener's workspace/immediate container. Unknown/null
  opener falls back to the active workspace's active page, not its structural
  selection. Singleton roots gain a tabbed wrapper; shells gain a leaf.
  Existing sibling weights are untouched; the new sibling copies the anchor's
  positive relative weight (avoids underflow from dividing tiny weights).
  Foreground opens reveal/activate/select the new leaf. Background opens retain
  workspace, active pages, structural selection and ancestor child/focus state;
  insertion into an empty shell necessarily initializes its active/selected leaf.
  A background split sibling is topologically visible without activation.
- `RevealTab(tab)` selects its workspace, active/structural leaf, every ancestor
  child along its path, and ancestor focus history, without retiling.
- `RemoveConfirmedTab(tab)` is the ONLY removal entry point; callers must use
  it after confirmed close/detach, never on close request or canceled unload.
  Removes the association and empty ancestor chain, preserving unary containers
  (including roots) and surviving weights. A removed selected child falls back
  to its next sibling, previous at end. Stale focus histories repair bottom-up
  through selected-child history. If the active leaf was removed, restore the
  nearest surviving parent's repaired remembered leaf and reveal its path.
  Surviving structural selection is retained; deleted selection becomes the
  repaired active leaf. Inactive-workspace repair never switches workspaces.
  Last removal leaves a null shell, including the final modeled tab: browser
  final-tab/window closure remains the future controller's responsibility.

All mutations reject invalid state/arguments without changing model state.
Multi-field mutations validate a candidate before committing. Parent links are
still derived; no duplicated ownership/topology cache was introduced.

## Verification executed

C/P/T/S abbreviations and replay algorithm are in implementation-workflow.md.
Starting C HEAD `ff3cd36042b53587dd45c50e1a3203a59d90584e`; T HEAD remains
`fbd20c49f1c3a8d0057a8714e5c9e98539cd2a21`, P HEAD remains
`2686b7c1cff1daccb5d5fbfa15c63a00fb236814`. Read full plan, scout, workflow,
T02/T03 results, CONTRIBUTING, and actual model/test sources before editing.

1. Preflight merged **and** canonical replay passed: **346 patches, 1,629
   footprint files, zero differences**, fuzz=0. C/T owned patches/series matched;
   P merged series matched S applied order. Empty unrefreshed delta at model top.
2. Final exported/reapplied actual-source **ASan + UBSan: 36/36 gtests passed**,
   no sanitizer findings. Includes all 25 earlier tests, table-driven opening
   dispositions/rejections and mutation sequences, and all **720 removal orders**
   of the six-tab recursive sample (Validate after each of 4,320 removals).
   Reproduce from S:

   ```bash
   third_party/llvm-build/Release+Asserts/bin/clang++ \
     -std=c++23 -Wall -Wextra -Werror \
     -fsanitize=address,undefined -fno-omit-frame-pointer \
     -I. -Iout/Default/gen \
     -Ithird_party/googletest/src/googletest/include \
     -Ithird_party/googletest/src/googletest \
     chrome/browser/ui/helium/workspaces/workspace_layout_model.cc \
     chrome/browser/ui/helium/workspaces/workspace_layout_model_unittest.cc \
     third_party/googletest/src/googletest/src/gtest-all.cc \
     third_party/googletest/src/googletest/src/gtest_main.cc \
     -pthread -o /tmp/i3-t04/model_tests
   /tmp/i3-t04/model_tests
   ```

   Uses actual Chromium typed IDs and vendored gtest, not stubs; focused binary
   uses the host standard library rather than the complete browser harness.
3. **Real Chromium production object build passed: 7 steps**, generated target
   flags, after platform reapplication/export:

   ```bash
   docker run --rm --network none --read-only --tmpfs /tmp \
     --user "$(id -u):$(id -g)" -e HOME=/tmp -e SCCACHE_DISABLE=1 \
     -v "$P:/repo:rw" -w /repo/build/src --entrypoint bash \
     chromium-builder:trixie-slim -c '
       third_party/siso/cipd/siso ninja -C out/Default -local_jobs 2 \
         obj/chrome/browser/ui/ui/workspace_layout_model.o'
   ```
4. **Full Chromium test target remains blocked, not passed.** Repeated command
   above with `obj/chrome/test/unit_tests/workspace_layout_model_unittest.o`:
   scheduling failed because existing prerequisite
   `chrome/test/data/extensions/extension_api_unittest/api_features.json` is
   missing, needed by `gen/chrome/common/extensions/extension_features_unittest.cc`.
   No casual repair/setup/download/reset, full unit_tests or browser run.
   This task is pure model code, not framework/runtime browser integration.
5. Final post-build/test replay again passed merged and canonical: **346 patches,
   1,629 files, zero differences**. Five owned source hashes matched after owned
   pop/push AND after final builds/tests. `git diff --check` passed.

### ISC evidence (model scope only)

- ISC-1: `WorkspaceCreationSwitchingAndShells` compares complete original state,
  verifies retained roots and last active/independent structural selection.
- ISC-2/3: `SplitWrapsExactLeafAndPreservesOuterWeights` covers both split axes,
  hidden nested insertion and singleton root wrapping. Earlier T03 tests pass.
- ISC-7: `RemovalRepairsHistoryButRetainsUnaryAndWeights`,
  `RemovalFromInactiveAndHiddenBranchesRetainsActivation`, and
  `AllRemovalOrdersRetainValidWorkspaceShells` cover active/hidden/inactive/final
  removals, empty ancestor chains, selection/focus repair, and retained shells.
- ISC-12: `ExternalRevealSelectsEveryAncestorWithoutRetiling` checks a hidden
  leaf under three tabbed ancestors across workspace selection and topology.
- Disposition/atomicity: `BackgroundInsertionPreservesFocusAndSelection`,
  `ForegroundOpenerInsertionRevealsDestination`,
  `MutationSequenceValidatesAfterEveryOperation`,
  `MutationRejectionsAreAtomicAndDoNotAllocate`, and
  `RefilledShellDoesNotRecycleRemovedNodeIds`.
- ISC-11/A-1/A-3 foundations: inspected API has no close-request mutation,
  WebContents/browser ownership, persistence, or transient tab indices. Actual
  canceled-beforeunload/browser lifecycle behavior is NOT verified by pure tests;
  it remains T09–T11/T14 work. No browser-level ISC readiness claim.

## Patch provenance and preservation

Used the verified container quilt wrapper: snapshot patches/.pc and owned source;
`pop -R helium/ui/workspaces/model.patch`; source edits only in registered files;
`refresh -p ab --no-timestamps --no-index --strip-trailing-whitespace`; empty
`diff -z`; save five source hashes; single owned `pop -R` / `push`; hashes match;
`push -a` platform suffix without force/refresh. No setup/reset/pull/broad copy.

Scratch unmerge `/tmp/i3-t04-export-5VW7lD` routed the owned patch into generic
output. Generic series matched C/T exactly; platform series matched series.orig.
Read the entire 1,438-line generated patch; copied only model.patch E -> T -> C
following prior-copy comparison. All P/T/C copies SHA256:

`e341a2d356c51696734da2c90d16e1c7508902d6a898e759941f4b67acd26185`

Source SHA256s (under chrome/browser/ui/helium/workspaces):

```text
33f7f19085943d98f58e4be52a551c0c461755c52326d68fb885804e66d58411  workspace_layout_model.h
f203dee9019b3d25029188dda0c7ae65f8b204425a2b968b616d4138900aa005  workspace_layout_model.cc
a52fa7e498a228f43b310e2abec1a0c0e4b88ff76727d3ec071e04fc3d69c1ab  workspace_layout_model_unittest.cc
```

All **693 other live patch/metadata files** in the archive remain byte-identical.
All 337 C/T generic patch pairs match; series unchanged; merged/applied order
matches, with model.patch before nine platform entries. No refresh churn exported.
Deleted AGENTS.md, dangling CLAUDE.md and empty build-local.log preserved. No
commit skill found; use CONTRIBUTING's scope-first commit and explicit staging.
Ephemeral snapshot/logs/commands: `/tmp/i3-t04/`, including preflight/final replay,
source diff/hashes, test/build logs, previous patch/series and source copies.

## T05 handoff

No model blocker. T intentionally remains dirty with owned model.patch and its
series entry; C is still canonical. Quilt top is the final Linux patch. T05 owns
new `helium/ui/workspaces/geometry.patch`, placed after model.patch and before
platform suffix via the documented workflow, NOT manual patch hunks.

Implement geometry against the unchanged value representation: positive finite
relative weights (possibly extreme, not normalized), ordered recursive n-ary
children, meaningful unary nodes, selected-child traversal for tabbed visibility,
active workspace/root and active leaf, and empty workspace shells. Structural
selection may be hidden and is not the active-page rectangle. Historical focus
may be hidden, but each nonempty workspace's active leaf always validates as
visible. Background insertion may add a visible split leaf without activation.
No geometry/focus-neighbor API was preemptively added in T04.

Browser creation/confirmation/reconciliation belongs to later controller tasks;
empty-shell switching itself never allocates browser tabs. Preserve the known
full test-target limitation and do not claim browser integration readiness.
