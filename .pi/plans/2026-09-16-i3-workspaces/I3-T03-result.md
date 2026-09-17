# I3-T03 — layout switching and structural selection

2026-09-16. **Accepted; T04 ready.** Only T03 implemented. Local file-backed
plan checkbox marked complete; preexisting untracked plan/scout remain unstaged.
This tracked result is the durable completion record.

## Delivered and API contract

Refreshed only `patches/helium/ui/workspaces/model.patch`, editing its existing
three model/test source files in S. Existing GN registration is unchanged.
Verified APIs against T02's actual value-state source and typed-ID generator;
no assumed browser/controller API was introduced.

- `SelectNode`, `SelectParent`, `SelectChild`: active-workspace structural
  navigation only, including hidden selections; never changes active page or
  container remembered child. Child navigation follows `selected_child`.
  Parent at root and child at leaf return false without changes.
- `ResolveLayoutTarget`: container itself, leaf's immediate parent, or singleton
  leaf root (the latter signals wrapping to SetLayout). Empty/invalid => null.
- `SetLayout(layout, external_node_generator)`: preserves descendants, IDs,
  order, associations, relative weights and historical focus. When converting
  an active ancestor to tabbed, selects the child containing the active leaf.
  Hidden targets do not activate/reveal themselves. Singleton leaf roots gain
  one meaningful unary container, selected structurally; subsequent layout
  commands reuse it. Existing generated-ID collisions are skipped. The caller
  must retain generators outside snapshots and never recycle removed IDs.
- `SetSelectedChild(container, child)` is **explicit child activation**, not
  structural descent: selects that child structurally, keeps the current active
  leaf if already within it, otherwise restores the child's remembered leaf.
  Reveals its ancestor path and updates ancestor focus history. This works for
  whole split subtrees and historical focus hidden behind tabbed descendants.
- Invalid state/arguments fail without mutation. Layout/activation changes use
  validated candidate copies. Parent relationships are derived, not stored.
  Empty shells stay null; no flattening, normalization, workspace creation,
  insertion/removal, browser ownership or T04 external-tab-reveal API added.

## Verification executed

Using C/P/T/S and container quilt wrapper from implementation-workflow.md:

1. Preflight merged **and** canonical source replay: **346 patches, 1,629
   footprint files, zero differences**. Starting C HEAD
   `1f8147911fce9b1c4928d99ed29a35e8c32fbb0d`; T HEAD remains
   `fbd20c49f1c3a8d0057a8714e5c9e98539cd2a21`, P HEAD remains
   `2686b7c1cff1daccb5d5fbfa15c63a00fb236814`.
2. Final exported/reapplied source: **25/25 focused gtests passed**, including
   all 17 T02 tests, with **ASan and UBSan, no findings**. Actual model/source,
   Chromium typed IDs and vendored gtest (not stubs) were compiled from S:

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
     -pthread -o /tmp/i3-t03/model_tests
   /tmp/i3-t03/model_tests
   ```

   This focused binary uses the host standard library, not the browser harness.
3. **Production object build passed: 7 steps**, using generated Chromium flags:

   ```bash
   docker run --rm --network none --read-only --tmpfs /tmp \
     --user "$(id -u):$(id -g)" -e HOME=/tmp -e SCCACHE_DISABLE=1 \
     -v "$P:/repo:rw" -w /repo/build/src --entrypoint bash \
     chromium-builder:trixie-slim -c '
       third_party/siso/cipd/siso ninja -C out/Default -local_jobs 2 \
         obj/chrome/browser/ui/ui/workspace_layout_model.o'
   ```
4. **Chromium test target still blocked**, not passed: same command with
   `obj/chrome/test/unit_tests/workspace_layout_model_unittest.o` fails scheduling
   because `chrome/test/data/extensions/extension_api_unittest/api_features.json`
   is missing, required by generated extension_features_unittest.cc. No repair,
   setup/reset/pull, full unit_tests or browser execution was performed. Pure
   model work introduces no runtime framework integration requiring a launch.
5. Owned source hashes match after pop/push and again after final build/tests.
   Final merged/canonical replay repeated after tests: **346 patches, 1,629
   files each, zero differences**. `git diff --check` passed. Local system
   clang-format used (bundled clang-format path was absent).

### Acceptance / ISC evidence (model scope only)

- ISC-2/3: `ChildActivationExposesWholeSplitSubtree` preserves both split leaves
  and nested layout after tabbed-child activation; `ChildActivationRevealsHistoricalHiddenFocus`
  and `LayoutOnHiddenContainerDoesNotRevealIt` test ancestor visibility.
  No claim about rendered rectangles: geometry/view tasks remain pending.
- ISC-4: `LayoutRoundTripPreservesExactNestedTopology` checks every field against
  an expected snapshot through tabbed/vertical/horizontal transitions, including
  nonuniform weights, inactive trees, leaf associations and remembered focus.
- ISC-5: `StructuralNavigationDoesNotActivatePages` and
  `LeafLayoutTargetsImmediateParent` verify independent activation, exact command
  targets and structural boundaries. `SingletonRootWrapsOnceAndRetainsUnaryNode`
  covers all three layouts, collision skipping and no redundant wrappers.
- `EmptyTargetsAndRejectedMutationsAreAtomic` checks empty shells/model, invalid
  IDs/layouts, foreign-workspace targets and cyclic input without state changes
  or ID consumption. All successful resulting states validate.

Browser-level ISC checkboxes remain pending; this is not feature exposure.

## Patch provenance and preservation

Snapshot at `/tmp/i3-t03/` includes patches/.pc archive, source-before, prior
owned patch/series, statuses, source diff, hashes and verification logs.
Popped only the nine platform suffix patches to the owned model.patch, refreshed
with `-p ab --no-timestamps --no-index --strip-trailing-whitespace`, verified
empty unrefreshed delta, then popped/pushed that patch and checked all five
owned source hashes. Reapplied platform suffix without refresh/force.

Scratch unmerge at `/tmp/i3-t03-export-cyTNQX` routed model.patch to generic
output; generic series exactly matched C/T, platform series matched series.orig.
Read entire exported patch before transferring only that file E -> T -> C.
All P/T/C copies SHA256:

`af9c8839117daad43c65d419491030514a55565a9e168a515ea6a1504c973ad4`

All **693 other live patch/metadata files** in the snapshot are byte-identical.
C/T series unchanged and all 337 generic patch pairs identical; S applied order
matches P's 346-entry merged series. Owned entry still precedes all platform
patches. No unrelated refresh churn exported. Deleted AGENTS.md, dangling
CLAUDE.md and empty build-local.log preserved. No commit skill was available;
used CONTRIBUTING.md scope-first message and explicit path staging.

## T04 handoff

No model blocker. T intentionally retains only owned patch/series changes;
canonical patch authority remains C. Live quilt top is the final Linux patch.
Pop to model.patch, follow the same narrow refresh/export/replay workflow.

T04 can add workspace/insertion/removal transactions on this value model.
Keep generators outside copied snapshots, retain positive finite relative
weights and meaningful unary nodes, validate historical descendant focus and
visible active leaf, and derive rather than persist parent links. Structural
selection may be hidden; explicit SetSelectedChild activation is distinct from
SelectChild navigation. Ancestor reveal currently exists only as part of child
activation; general external-tab selection/workspace switching is still T04.
Complete Chromium test execution remains blocked by the existing missing fixture;
no browser/integration readiness is claimed.
