# I3-T02 — tree types and invariant tests

2026-09-16. **Accepted; T03 ready.** Only T02 implemented. Local file-backed
plan checkbox marked complete; the preexisting untracked plan/scout are not
staged. This tracked result is the durable completion record.

## Delivered

`patches/helium/ui/workspaces/model.patch` adds three files under
`S/chrome/browser/ui/helium/workspaces/` and registers production sources in
`//chrome/browser/ui:ui` and tests in desktop `//chrome/test:unit_tests`.

- `WorkspaceLayoutModel`: pure value forest, ordered workspaces, n-ary ordered
  horizontal/vertical/tabbed containers, leaves with opaque runtime tab keys.
- Distinct `base::IdTypeU64` node/workspace/tab IDs; null IDs represent empty
  workspace shells. Use each type's Generator and never recycle IDs.
- Structural selection is separate from each workspace's active/last-active
  leaf. Containers retain selected child, relative weights, last-focused leaf.
- Iterative traversal validates the whole forest, including hidden/inactive
  branches; rejects cycles, shared children/roots, orphans, missing/null IDs,
  duplicate tab associations, invalid selection/activation, selected-child
  nonmembership, bad remembered focus, invalid weights/layouts/workspaces.
- No Views, WebContents, tab ownership, strip indices, persistence, commands,
  mutations, or browser feature exposure. No T03/T04 implementation.

Verified local conventions: `base/types/id_type.h` supports typed generators,
map keys and null checks; `TabInterface` inherits `SupportsTabHandles` and
exports `tabs::TabHandle` (tab_interface.h:360). `tab_handle_factory.h` keeps
session mappings separate from handles. The future controller maps TabKey to
those handles and checks live-tab coverage; the pure validator cannot do that.
GN source placement was inspected in the real `ui` static library and desktop
`unit_tests` source list; tests use Chromium's gtest wrapper. Helium copyright
headers match the existing layout controller.

## Verification (actual execution)

All paths below use C/P/T/S from `implementation-workflow.md`.

1. **Preflight replay passed** before edits: merged and canonical stacks each
   applied 345 patches with fuzz=0; all 1,626 footprint files matched S.
   Starting C HEAD `da922d0e0a02c25e2d08fb792d40d588cdf0d2a7`; T remained at
   `fbd20c49f1c3a8d0057a8714e5c9e98539cd2a21`, P at T01's recorded revision.
   GN hashes exactly matched T01. New workspace directory was absent.
2. **GN generation passed**: `cd "$S"; buildtools/linux64/gn gen out/Default`
   reported `Done. Made 32686 targets from 4982 files in 10043ms`.
3. **Actual Chromium production object build passed**, after platform reapply:

   ```bash
   docker run --rm --network none --read-only --tmpfs /tmp \
     --user "$(id -u):$(id -g)" -e HOME=/tmp -e SCCACHE_DISABLE=1 \
     -v "$P:/repo:rw" -w /repo/build/src --entrypoint bash \
     chromium-builder:trixie-slim -c '
       third_party/siso/cipd/siso ninja -C out/Default -local_jobs 2 \
         obj/chrome/browser/ui/ui/workspace_layout_model.o'
   ```

   Result: `Build Succeeded: 7 steps`, including compilation of the new model
   using Chromium's generated target flags. No args.gn modifications. Existing
   out/ build metadata/generated outputs were updated by GN/siso normally.
4. **Focused standalone tests passed, 17/17**, compiling the actual source,
   Chromium typed-ID headers and vendored GoogleTest, not stubs. Repeated on
   final exported/reapplied source with AddressSanitizer and UBSan: 17/17,
   no sanitizer findings. Reproduce from S (omit sanitizer flags for plain run):

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
     -pthread -o /tmp/workspace_model_tests
   /tmp/workspace_model_tests
   ```

   This focused executable uses the host C++ standard library, not the complete
   browser test harness. The separate production object build above covers
   Chromium toolchain compilation of the model.
5. **Full Chromium unit-test target blocked, not passed.** Host depot_tools
   autoninja first failed before building: `python3_bin_reldir.txt not found`.
   Direct container siso with both real object paths (model above and
   `obj/chrome/test/unit_tests/workspace_layout_model_unittest.o`) reached GN
   but could not schedule the test target because an existing prerequisite is
   missing: `chrome/test/data/extensions/extension_api_unittest/api_features.json`,
   needed by `gen/chrome/common/extensions/extension_features_unittest.cc`.
   No setup/download/reset was used to repair the source tree. Initial autoninja
   invocation also used incorrect dot-separated object names; generated Ninja
   inspection established the slash-separated paths used by direct siso.
   Full `unit_tests --gtest_filter=WorkspaceLayoutModelTest.*` and browser tests
   were **not run**. No browser launch required for this pure-data-only task.
6. **Final replay passed again after build/tests:** both merged and canonical
   replay: 346 patches, 1,629 files, zero differences; all five owned source
   SHA256s unchanged after owned-patch pop/push and final build.

Named tests cover the T02 ISC foundations: `RecursiveForestAndIndependentSelection`
proves mixed recursive n-ary split/tabbed representation and distinct selection
(ISC-2/3/5); `UnaryContainersAreMeaningful` preserves singleton structure.
Malformed cases include `RejectsCyclesIncludingHiddenSubtrees`,
`RejectsDuplicateAndNullAssociations`, `RejectsNonChildSelectionForEveryLayout`,
`RejectsSelectionOutsideWorkspace`, `RejectsInvalidOrHiddenActiveLeaf`, and
`RejectsSharedRootsAndInvalidEmptyShells`. Header/implementation inspection
establishes no browser ownership or transient indices (A-1/A-3 foundations).
These are **foundations only**, not completion of browser-level ISC behavior.

## Patch provenance and preservation

Used the T01 container quilt wrapper and exact source-first gates: snapshot;
pop nine platform patches to exclusive-access-bubble; new owned model.patch;
add all five paths before edits; narrow refresh with `-p ab --no-timestamps
--no-index --strip-trailing-whitespace`; empty `q diff -z`; save owned hashes;
pop/push owned patch and compare; reapply platform suffix without refresh/force.
Scratch unmerge routed the patch to generic output. Its generic series equaled
prior canonical series plus exactly the owned line; platform series equaled
series.orig. Inspected entire generated patch before copying E -> T -> C.

All three copies (P/T/C) of model.patch have SHA256:
`c04e094701b04da1f025b08ed3a44a47a83a80558d8a79edc5382752a2a156af`.

C/T now have 337 generic patches; P/S have 346 merged/applied patches. The owned
entry is immediately after `helium/ui/exclusive-access-bubble.patch`, before
all nine platform entries. S applied-patches equals P series.merged. All 692
preexisting live patch/metadata files in the snapshot (excluding intentionally
changed series.merged) are byte-identical; all original 336 C/T patch pairs
still match. Existing generic refresh churn was not exported. `git diff
--check` passed. Deleted AGENTS.md, dangling CLAUDE.md and empty build-local.log
remain untouched. No whole-directory sync or destructive tooling was used.

Ephemeral logs/snapshot: `/tmp/i3-t02/` (quilt-and-patches.tar.gz, source.diff,
source-sha256, preflight/final/post-build-replay.log, GN/siso and test logs).
Scratch export: `/tmp/i3-t02-export-cfUkNO`. Workflow contains the durable
replay algorithm; commands/results above do not require those logs to survive.

## T03 handoff

T remains intentionally dirty with only owned model.patch and its series entry;
do not reset/pull it. Live quilt top is the final Linux disable-tab-strokes
patch. Pop to model.patch using T01 wrapper before adding T03 source edits;
refresh/export only that owned patch, and repeat all verification gates.

Add layout/selection mutations to this value-state foundation. Keep generators
outside copied candidate snapshots so failed mutations cannot recycle IDs.
Weights are positive finite relative values, not required to sum to one.
Container last-focused state must name a descendant leaf; it is historical and
need not equal the workspace active leaf. Active leaf must be visible along
all tabbed ancestor selections. Empty shell selection/active IDs must be null.
Unary containers are valid and should not be flattened. Structural selection
may reference hidden nodes; page activation remains separate. No parent map is
stored; T03 can derive it, avoiding duplicated structural state.

No T03 model blocker. Complete Chromium unit-test execution remains blocked by
the missing existing fixture/depot_tools bootstrap; keep that limitation visible
and do not claim full browser/integration readiness.
