# I3-T07 — native subtree selectors and structural selection

2026-09-16. **Implementation complete under the user's revised validation
policy. Runtime acceptance deferred, NOT passed.** T08 implementation may proceed.
T06 remains a compiled, isolated foundation with pending runtime acceptance.
No tests, browser launch, resource restoration or downloads were performed.
See tracked `validation-status.md`; the local preexisting untracked plan received
only a validation note and T07 implementation status, and remains unstaged.

## Implementation and boundaries

Only `patches/helium/ui/workspaces/views.patch` refreshed/exported. Added native
`workspace_tabbed_header_view.{h,cc}` to the existing desktop `:ui` target;
extended the existing host and browser-test source. No model/geometry changes,
new patch/series entry, BrowserView routing, controller, resize implementation,
commands or second tab/WebContents owner.

- One native LabelButton for each **immediate child** of each visible tabbed
  container. `Tabbed[Email, Split(A,B)]` has two selectors. Nested visible tabbed
  containers obtain separate headers from geometry; hidden containers do not.
- `kTabList`/`kTab` accessibility roles, explicit selected state, accessible
  names, native pointer/default AX actions and button keyboard focus behavior.
  A three-DIP bottom rule indicates selected child. Native button focus remains
  separate. Controls shrink/elide in their allocated native header rectangle.
- Leaf labels use caller-supplied TabKey titles. Subtree labels use the remembered
  descendant's title prefixed with `Layout:`. Missing/control-only titles fall
  back to `Untitled page`; text is capped at 160 UTF-16 code units before shaping,
  strips control/bidi-override characters and avoids truncating a surrogate pair,
  then applies Chromium's bidi sanitization. No markup/URL fallback or live page
  observer. These isolated-harness English labels need resource/localization
  treatment before production exposure.
- `UpdateTitles(Titles)` updates text, tooltip and AX name without detaching pages
  or changing selected child. Supply a complete current title map, including
  hidden children; absent entries deliberately use fallback rather than retaining
  stale titles. Topology/remembered-leaf changes require a new presentation.
- A four-DIP square outline marks the **structurally selected visible subtree**,
  unlike native rounded page focus or the header's selected-child bottom rule.
  Bounds are the union of its displayed leaves and headers, covering intervening
  split space. Hidden selection has no outline; constrained projection suppresses
  container selection rather than pretending its full subtree is displayed.
  Selection calculation never substitutes active_leaf or changes activation.
  Outline is ignored by AX and does not intercept input. Native colors are theme
  tokens; actual contrast/paint-order/focus UX remains runtime-unverified.
- Shared physical-edge-to-DIP conversion is used for headers, leaves and outline.
  Empty projections clear headers/outline. Invalid presentations preserve the
  existing host, as before. `GetHeaders()` and `selection_bounds()` expose narrow
  inspection contracts, not ownership; header/container pointers expire on
  successful Present/Clear.

### Input safety and T09 callback contract

Reviewed T06 audit and actual host API, model mutation behavior, native
TabbedPane accessibility patterns, LabelButton/Button callbacks, View's
`kDoDefault` mouse dispatch and BoxLayout shrink policy.

`WorkspaceTabbedHeaderView::SelectChild` is a synchronous request callback:
`void(NodeId container, NodeId immediate_child)`. `Update` validates the full
model, tabbed kind and visibility in the active workspace, then copies only
stable IDs/title associations. No page/model pointers are retained. Dispatch
checks membership in the presented immediate-child set. Weak button callbacks
expire on Update/destruction; callback is copied before dispatch and no members
are accessed afterwards, permitting synchronous owner reconciliation/deletion.
Invalid/hidden/non-tabbed Update requests are no-ops, preserving the old surface.

**A view snapshot is not authority over a newer model.** The future controller
must bind a weak owner and presentation/workspace generation, revalidate current
IDs and lifecycle state, call SetSelectedChild through its mutation boundary,
reconcile actual browser activation/visibility and present the committed result.
The view does not optimistically select/reveal or activate pages itself. T09/T10
must supply title-change notifications and activation/foreground adapters.

**No events were enabled on the incomplete host.** WorkspaceContentsView still
calls `SetCanProcessEventsWithinSubtree(false)` and constructs headers with a
null callback (disabled buttons). Input-capable headers have an explicit safe
boundary: standalone native surfaces with **no WebContents/page hosts**. The new
test specifies this boundary. A display-only host header cannot issue a mutation
even through a direct AX invocation. Existing T06 native-input/focus caveats
remain; do not attach real browser tabs or lift suppression to try the UI.
Feature remains disabled by default, with no production construction path.

## Compilation and static evidence (NOT runtime verification)

Starting C HEAD: `e3fcbc1d09f2f9d2828892946d0c2ef3ddc78af3`.
T/P revisions remain those recorded in T06. C/P/T/S path definitions and quilt
wrapper are in `implementation-workflow.md`. Evidence/snapshots are ephemeral
under `/tmp/i3-t07/`.

1. Read CONTRIBUTING, plan, scout, workflow, T05/T06 results and build-blocker
   report. No commit skill available in the advertised/local skill directories;
   used CONTRIBUTING's scope-first convention and explicit staging.
2. Preflight merged/canonical replay: **348 patches / 1,635 files**, fuzz=0,
   zero differences. Saved patches/.pc archive, existing owned source copies,
   GN file and P/T statuses. Empty unrefreshed delta at views.patch top.
3. **Production objects compiled successfully**, including final exported,
   platform-reapplied state: GN generated 32,685 targets / 4,981 files; siso
   completed **8 steps**, recompiling both host and header. Command:

   ```bash
   docker run --rm --network none --read-only --tmpfs /tmp \
     --user "$(id -u):$(id -g)" -e HOME=/tmp -e SCCACHE_DISABLE=1 \
     -v "$P:/repo:rw" -w /repo/build/src --entrypoint bash \
     chromium-builder:trixie-slim -c '
       buildtools/linux64/gn gen out/Default &&
       third_party/siso/cipd/siso ninja -C out/Default -local_jobs 2 \
         obj/chrome/browser/ui/ui/workspace_contents_view.o \
         obj/chrome/browser/ui/ui/workspace_tabbed_header_view.o'
   ```

4. **Updated native browser-test object compiled, not linked/run.** Obtained
   the generated command with:

   ```text
   third_party/siso/cipd/siso query commands -C out/Default \
     obj/chrome/test/browser_tests/workspace_contents_view_browsertest.o
   ```

   Extracted only the command compiling that exact source. Retained generated
   flags/includes/modules/compiler/sysroot; changed only output prefix from
   `obj/chrome/test/browser_tests/workspace_contents_view_browsertest` to
   `/e/workspace_contents_view_browsertest`. Executed in the same container at
   `/repo/build/src/out/Default`, mounting `/tmp/i3-t07:/e`, SCCACHE_DISABLE=1.
   Final exit 0, no diagnostics, object **477,792 bytes**. Saved script
   `compile-test.sh`, logs `test-object-final.log` and `verified-build.log`.
   Initial compile caught a capturing lambda passed to BindRepeating; corrected
   to the repository's BindLambdaForTesting helper. Query output included binary
   resource command text; extraction decoded unrelated lines with replacement
   but asserted the selected compiler command contained no replacement bytes.
   No generated build graph, inputdeps, GN arguments or resources were modified.
5. Tests authored/extended, **NOT RUN**:
   - Existing `LiveLeavesAndSelectedSplitSubtree`: header child identities,
     display-only controls, title updates without reattachment, full-container
     versus hidden structural selection without active-page changes, empty clear.
   - `ImmediateSubtreeSelectorsAndNativeActions`: two selectors for three leaves,
     subtree label, native mouse and AX request IDs, no optimistic mutation,
     owner-driven selected AX state, missing/control/long-title handling, invalid
     and hidden container rejection. Uses a standalone contents-free header.
   - Static review corrected a test's reference to children across model's
     copy/commit mutation to a value snapshot.
6. `git diff --check` passed. Owned source hashes match after pop/push, platform
   reapplication and final compile. Final and **post-build** merged/canonical
   replays each: **348 patches / 1,637 files**, fuzz=0, zero differences.

**Tests NOT RUN:** no model regression executable, unit_tests, browser_tests,
interactive suite or browser process was executed. Full native test target still
has the preexisting missing-resource blocker; no attempt to restore or bypass
it. Compilation alone does not satisfy native runtime acceptance, ISC-3/5/12,
or ordinary-window regression requirements. It supports their implementation
foundations only. No new runtime ISC checkbox is marked passed.

## Patch provenance / preservation

Used safe container quilt: pop only suffix to views.patch; register both new
sources **before creation**; edit S; refresh only views.patch with `-p ab
--no-timestamps --no-index --strip-trailing-whitespace`; inspect whole generated
patch and subsequent single test-reference correction; empty unrefreshed diff;
hash seven owned paths; owned pop/push and matching hashes; platform push -a
without force or refresh. Scratch unmerge verified unchanged C/T generic series
and original platform series; exported **only views.patch** E -> T/C.

P/T/C views.patch SHA256:
`4fcf452a0bb8d9731dfe9e8a9bff1c587f99e797083c68329afd7f728be18895`.

Final source SHA256 (under chrome/browser/ui/views/frame):

```text
d751e609398b830e00ce0ce23b66bc46aac98b41926774c25d3434aa8ba47b2b  workspace_contents_view.cc
9407fe93530b583ed53384f6eb8f1915837860abb1f5744712c2cf67c27a2ee6  workspace_contents_view.h
ef2d391ef9d37fa8b1a6eb04c3f6b1871eca4c3212b41a24b1ed5ac183409a13  workspace_contents_view_browsertest.cc
cda3a090000c769f60f9506b0712f8f11653595629ee238f8d56d1a622f24c83  workspace_tabbed_header_view.cc
6df9af16672be24d09cb9a13239f173bdeda428e60edf2d54226c51e140dbeaa  workspace_tabbed_header_view.h
```

Of **696** preexisting P patch/metadata files, only views.patch changed; all
other hashes match the archive. All **339** C/T generic patch pairs match.
Merged/applied and C/T series match; no series edit. P/T status lists unchanged.
Deleted AGENTS.md, dangling CLAUDE.md, build-local.log (empty SHA256 unchanged),
untracked scout/plan/blocker and unrelated platform/tooling state preserved.
No setup/reset/pull/broad sync, test downloads or resource restoration.

## T08 readiness and remaining risks

**T08 implementation ready under validation-status.md.** Continue source-first
in views.patch for divider surfaces, with model/geometry mutations only in their
owned patches if required by T08. Headers are selectors, not drag dividers.
Do not enable the host's subtree input to exercise pointer resizing: use an
isolated contents-free surface until lifecycle/controller integration exists.
No T08 resizing or T09 controller was implemented here.

Runtime native hosting, painting/contrast, tiny-header usability, pointer/AX
interaction, localization, focus restoration, live title observation and ordinary
native-split regression remain unverified. Header callbacks deliberately stop at
validated requests; controller generation/lifecycle revalidation and real page
activation/foreground remain T09/T10. All T06 primary AX, dialog, dynamic host
subscription, DevTools/fullscreen and ownership-transfer caveats remain. Keep
feature exposure off; T06/T07 runtime acceptance must be explicitly revisited in
an authorized validation phase, not inferred from successful object builds.
