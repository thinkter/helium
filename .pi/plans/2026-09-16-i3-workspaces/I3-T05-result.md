# I3-T05 — recursive workspace geometry and displayed leaves

2026-09-16. **Accepted; T06 ready.** Only T05 implemented. Local file-backed
T05 checkbox completed; preexisting untracked plan/scout remain unstaged.
This tracked result records completion without adopting unrelated files.

## Delivered / API contract

New `patches/helium/ui/workspaces/geometry.patch` adds
`workspace_layout_geometry.{h,cc}` and `_unittest.cc` under
`chrome/browser/ui/helium/workspaces/`, registering production sources in
`//chrome/browser/ui:ui` and tests in desktop `//chrome/test:unit_tests`.
Model patch/source remain unchanged. No Views/controller work, activation,
WebContents ownership, persistence, or feature exposure.

- `CalculateWorkspaceGeometry(model, pixel_width, pixel_height, metrics)` is a
  const projection of the active workspace, including valid empty shells.
  Invalid model/metrics or negative viewport dimensions return `valid=false`.
- Viewport and output rectangles are **physical pixels**, local to the content
  viewport's top-left. Minimum leaf dimensions, header height and divider size
  are supplied in integer DIPs; finite positive device scale converts them with
  ceiling once. Defaults are geometry policy inputs, not audited host sizing.
- Iterative bottom-up minima and top-down layout avoid recursive call-stack
  growth. Horizontal splits sum widths/max heights; vertical splits sum
  heights/max widths; each tabbed node adds its header height to only its
  selected child's minimum. Hidden children/workspaces never contribute.
  Unary splits have no divider; unary tabbed nodes retain their header cost.
  Minima saturate at INT_MAX+1, meaning no supported viewport can fit them.
- N-ary weighted allocation clamps undersized children to their recursive
  minima and redistributes among remaining children. Normalize by the largest
  remaining weight, avoiding overflow for DBL_MAX sums and tolerating denormal
  weights. Floor shares, then assign largest fractional remainders first with
  stable tree-order ties. Reserve minima before rounding to protect them from
  floating-point accumulation error. Integer extents exactly conserve space.
- Horizontal geometry mirrors in RTL; leaf ordering and rounding ties remain
  depth-first tree order. Header/divider rectangles are separate from page
  rectangles. Dividers identify their container and preceding tree-child index.
- Explicit `displayed_leaves` equals the positive-area leaf rectangle IDs.
  In constrained layouts, project **active_leaf**, never structural selection,
  over the viewport; suppress all headers/dividers. Zero-area viewport displays
  nothing. Resizing back recomputes the original tree; topology never changes.
- `FindDirectionalNeighbor` uses physical doubled centers to select the strict
  requested half-plane. Positive perpendicular overlap ranks before distance;
  then squared center distance, then existing tree order. Touching edges alone
  do not overlap. No wrap; missing/hidden sources and invalid output return null.

## Verification executed

C/P/T/S abbreviations and reproducible replay algorithm are in
`implementation-workflow.md`. Starting C HEAD
`fb9a8388468101f40df6df3b8ddb1e8aa933338d`; T HEAD
`fbd20c49f1c3a8d0057a8714e5c9e98539cd2a21`; P HEAD
`2686b7c1cff1daccb5d5fbfa15c63a00fb236814`.
Read full plan, scout, workflow, T02–T04 results, CONTRIBUTING, model header/test
patterns and actual MultiContentsView layout/minimum-size seams.

1. **Preflight merged and canonical replay passed:** 346 patches, 1,629 files,
   zero differences, fuzz=0. C/T series/model copies matched and P merged series
   equaled S applied order. Saved statuses, patches/.pc archive and both GN files
   before source edits. Empty unrefreshed delta at the model patch top.
2. **Final actual-source ASan + UBSan: 52/52 gtests passed**, no findings:
   36 unchanged model tests plus 16 geometry tests. Reproduce from S:

   ```bash
   third_party/llvm-build/Release+Asserts/bin/clang++ \
     -std=c++23 -Wall -Wextra -Werror \
     -fsanitize=address,undefined -fno-omit-frame-pointer \
     -I. -Iout/Default/gen \
     -Ithird_party/googletest/src/googletest/include \
     -Ithird_party/googletest/src/googletest \
     chrome/browser/ui/helium/workspaces/workspace_layout_model.cc \
     chrome/browser/ui/helium/workspaces/workspace_layout_geometry.cc \
     chrome/browser/ui/helium/workspaces/workspace_layout_model_unittest.cc \
     chrome/browser/ui/helium/workspaces/workspace_layout_geometry_unittest.cc \
     third_party/googletest/src/googletest/src/gtest-all.cc \
     third_party/googletest/src/googletest/src/gtest_main.cc \
     -pthread -o /tmp/i3-t05/workspace_tests
   /tmp/i3-t05/workspace_tests
   ```

   Actual Chromium typed IDs and vendored gtest, no stubs. Focused executable
   uses host standard library, not the full browser test harness.
3. **Chromium production object compilation passed: 7 steps**, final exported
   and platform-reapplied state, with actual generated production flags:

   ```bash
   docker run --rm --network none --read-only --tmpfs /tmp \
     --user "$(id -u):$(id -g)" -e HOME=/tmp -e SCCACHE_DISABLE=1 \
     -v "$P:/repo:rw" -w /repo/build/src --entrypoint bash \
     chromium-builder:trixie-slim -c '
       buildtools/linux64/gn gen out/Default
       third_party/siso/cipd/siso ninja -C out/Default -local_jobs 2 \
         obj/chrome/browser/ui/ui/workspace_layout_geometry.o \
         obj/chrome/browser/ui/ui/workspace_layout_model.o'
   ```

   GN generation passed (32,686 targets/4,982 files). Final verification requested
   both objects; geometry recompiled, existing model was already up-to-date.
   An earlier attempt generated GN on the host before platform reapplication;
   subsequent container build failed in uBlock's generator on host-absolute
   output paths. Reapplying the suffix and regenerating GN **inside the same
   container mapping** resolved it. No source repairs or args.gn edits.
4. **Full Chromium test target remains blocked, not passed.** Same container
   siso command targeting
   `obj/chrome/test/unit_tests/workspace_layout_geometry_unittest.o` fails
   scheduling: missing existing
   `chrome/test/data/extensions/extension_api_unittest/api_features.json`, needed
   by `gen/chrome/common/extensions/extension_features_unittest.cc`. No casual
   fixture repair/download/reset. Full unit_tests/browser tests were not run;
   pure geometry work adds no framework integration requiring a browser launch.
5. **Post-build replay passed:** merged and canonical each 347 patches, 1,632
   files, zero differences. All five owned source hashes match after owned
   pop/push and after final builds/tests. `git diff --check` passed.

### Acceptance / ISC evidence (geometry scope only)

- ISC-2/3: `MixedNestedLayoutAndExplicitDisplayedSet` verifies weighted
  H(leaf, Tabbed(V(leaf,leaf), hidden), leaf), exact leaf/header/divider bounds,
  recursive minima and a selected whole subtree. `TabbedMinimaFollowOnlySelectedSubtree`
  verifies hidden subtree exclusion and header-driven minimum transitions.
- ISC-8 foundation: explicit displayed set checked against every returned leaf
  rectangle by `ExpectPartition`. That helper checks nonnegative bounds,
  viewport containment, no overlap, and exact viewport area coverage. Browser
  foreground reporting remains T10; this is not a claim about live renderers.
- `ConstrainedUsesActiveNotHiddenSelection` tests each axis below minimum,
  1x1, zero-area projection, and restoration without topology/weight changes.
  `WorkspaceShellAndSingleton` and `UnaryAndNestedHeaderCosts` cover shells,
  inactive workspaces, unary split/divider suppression and nested headers.
- Weighted tests cover clamping, unequal shares, fractional remainders,
  stable ties on both axes/RTL, DBL_MAX and denormal relative weights.
  `FractionalDPIAndMirroredNestedGeometry` verifies 1.25 scale and mirroring.
  `SaturatedMinimaAndMaximumViewport` covers INT_MAX extents and extreme DPI.
- `DeepUnaryTreesDoNotUseRecursiveCallStack`: 2,048-node chain.
  `SeededWeightAndViewportSweepPartitionsExactly`: 500 reproducible cases with
  1–12 siblings, weights spanning exponents -1000..999, both axes, RTL, DPI and
  varied viewports, checking partition conservation and minimum bounds.
- Directional tests cover all four physical directions, edges/no wrap, hidden
  source, constrained projection, overlap preference over nearer diagonal,
  non-overlap distance, center-half-plane exclusion and stable tree-order ties
  independent of numeric IDs. Invalid input rejection is separately tested.

## Patch provenance and preservation

Used the verified container quilt wrapper: snapshot; pop platform suffix to
model.patch; `new helium/ui/workspaces/geometry.patch`; register all five files
before editing; source-first edits; `refresh -p ab --no-timestamps --no-index
--strip-trailing-whitespace`; empty `diff -z`; save hashes; owned pop/push and
hash comparison; platform `push -a` without force/refresh. Never hand-authored
diff hunks. System clang-format used on the three new C++ files.

Read the entire generated patch before initial export; inspected the final
one-line include addition and hunk count adjustment before final export.
Scratch unmerge `/tmp/i3-t05-export-final-xa_a6ssr` routed geometry.patch into
its generic output. Generic series matched C/T, platform series matched
series.orig. Only geometry.patch transferred E -> T -> C; only its approved new
series line appended. All three P/T/C patch copies SHA256:

`e80a63ce0f85edc27de30d6451ad65bece222ebff486fc4572e3b9eb50d42d1b`

Source SHA256s under chrome/browser/ui/helium/workspaces:

```text
d3c20118798fbf606469349cbeb84147b3f5c882af440536a1dfd7c6c22baf3a  workspace_layout_geometry.h
94fc2ef671358e7bf789cb2559927827bb8a4b4361f31976c531f4075f41f30b  workspace_layout_geometry.cc
03fb049a82fd4778f21e8412f5283699ad0c5daf9bbb9f462e64758a0197af08  workspace_layout_geometry_unittest.cc
```

All **693 preexisting live patch/metadata files**, excluding intentionally
changed series.merged, remain byte-identical to the snapshot. All **338 C/T
generic patch pairs** match. Geometry follows model, before all nine platform
entries; merged/applied series match. Model patch remains T04 SHA256
`e341a2d356c51696734da2c90d16e1c7508902d6a898e759941f4b67acd26185`.
Deleted AGENTS.md, dangling CLAUDE.md and build-local.log preserved. No broad
copy into real checkouts, reset/setup/pull, or unrelated patch refresh/export.
No commit skill found; follow CONTRIBUTING scope-first style and explicit paths.
Ephemeral evidence/snapshots/logs live in `/tmp/i3-t05/`.

## T06 handoff

No geometry blocker. C remains canonical; T intentionally retains owned
model/geometry patches and series edits. Quilt top is the final Linux patch.
Create views.patch after geometry.patch using the same source-first gates.

Host must audit primary-view/focus/dialog/visible enumeration contracts before
integration. Convert viewport DIPs into the documented physical-pixel boundary,
then convert shared rectangle edges back consistently for Views (do not round
individual widths independently or mirror RTL twice). Supply measured host
minimum/header/divider DIPs; defaults here are not promises of final native
header styling. Existing MultiContentsView's percentage-based pair clamp is
not recursive and was deliberately not imported as the tiling policy.

Use explicit displayed_leaves for eventual attachment/foreground reconciliation;
zero-area means none. Use constrained to show the future constrained-layout
indicator without changing the tree. Resolve NodeIds through the model/controller,
never store WebContents in geometry. Neighbor results are proposed leaf targets,
not browser activation. Structural selection can be hidden and remains separate.
T06/T07 own view placement/headers/selection feedback; T08 owns resize mutation;
T09/T10 own actual lifecycle/activation/foreground behavior. No browser-level
ISC completion or integrated feature readiness is claimed.
