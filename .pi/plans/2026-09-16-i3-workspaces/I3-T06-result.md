# I3-T06 — native content-host audit and isolated rendering checkpoint

2026-09-16. **Partial checkpoint; T06 remains unchecked.** Production and native
browser-test objects compile. Native test execution is blocked by the existing
missing test fixture; displaying live leaves and ordinary-window regression
acceptance have NOT passed. No browser mode is exposed. T07/T09/T10 were not
implemented.

## Contract audit (actual applied source)

Paths below are relative to S from implementation-workflow.md. Read the approved
plan/scout/workflow, T05 result, CONTRIBUTING, model/geometry APIs, native host
implementations and existing test patterns before source edits.

- **Construction:** `chrome/browser/ui/views/frame/contents_container_view.cc:62`
  requires a live BrowserView, profile and browser services. It creates a page
  ContentsWebView, hidden DevTools WebView, modal host, toast anchor, scrims,
  outlines/capture borders, and feature-dependent footer, reading, actor, Glic
  and other overlays. This is not a profile-only generic page container. Its
  destructor explicitly removes reading overlay before its referenced page.
  The host must die before BrowserView. Dynamic construction alone does not
  register all browser controllers.
- **Primary page is not ownership or activation:** MultiContentsView initially
  designates slot zero primary (`multi_contents_view.cc:90`); SetActiveIndex
  toggles primary true/false on the pair (`:336`). WebView's flag defaults false.
  In `ui/views/controls/webview/webview.cc:467`, the legacy accessibility path
  uses it to mark the renderer AX platform delegate primary for the window;
  ViewsAX instead uses child-tree embedding. Merely setting it does not activate
  a tab. Never mark every dynamic page primary, or make the separate test host
  another primary in the ordinary browser's window. This checkpoint sets none.
- **Attachment:** MultiContentsView::SetWebContentsAtIndex (`:268`) attaches via
  ContentsWebView and separately reinstalls SadTabHelper. CloseSplitView detaches
  inactive contents and restores the surviving sad tab. ContentsWebView's
  SetWebContents updates blocked state, creates/resets its status bubble, and
  asks an existing WebContentsModalDialogManager to update the dialog host.
  It does NOT replace BrowserView's per-page host lookup. The experimental host
  explicitly detaches old pages before attaching a new projection and reinstalls
  sad tabs when a helper exists; no WebContents ownership enters the model.
- **Focus:** MultiContentsView (`:135–168`, `:493–540`) installs scoped page,
  footer, actor and reading-overlay focus subscriptions plus a class-name map.
  Its delegate (`multi_contents_view_delegate.cc:29–49`) activates only tabs
  belonging to the active native pair, guarding stale pair callbacks. This
  cannot activate arbitrary workspace leaves. BrowserView's
  MaybeUpdateStoredFocusForWebContents (`:5050`) remaps stored focus through
  that pair's map. ContentsWebView also requests focus on Linux mouse/touch
  interaction. Views event handling suppression is NOT a guarantee against
  renderer/native focus; no interactive browser use is allowed in this slice.
- **Dialogs:** BrowserView::GetWebContentsModalDialogHost (`:4067`) looks up the
  page only in MultiContentsView, falling back to its active container.
  UpdateTabModalDialogHost (`:5034`) uses MultiContentsView's visible iteration
  and real TabFeatures. `tab_modal_dialog_host.cc` anchors to BrowserView's widget
  and toolbar, computes maximum size from the whole contents area, and special
  cases only the bottom native stacked pane via `contents_container_views().back()`.
  Reusing this modal object does not establish arbitrary leaf dialog positioning.
  Tests here deliberately use independent pages without browser TabFeatures or
  modal dialogs; this is not a test of browser-owned-tab lifecycle.
- **Enumeration is still pair-specific:** BrowserView::GetAllVisibleContentsWebViews
  (`:5080`) returns active plus inactive iff IsInSplitView. Consumers include
  GetStatusBubbles (`:2482`), toolbar animation layout invalidation (`:3067`),
  non-Linux drag completion (`:3094`), native hosts for top-controls sliding
  (`:4393`) and fast-resize propagation (`:6145`). Some additionally directly
  lay out MultiContentsView: changing just the vector-returning helper is not
  sufficient.
- **Active/all-host consumers:** BrowserView's active, per-contents and all-
  container getters (`:1372–1383`) delegate directly to MultiContentsView.
  GetContentsSize (`:2795`), drag coordinate conversion (`:3933`), modal fallback,
  stored-focus remapping and split-update paths rely on these. Accessible F6
  traversal (`:5169`) directly uses MultiContentsView::GetAccessiblePanes;
  ContentsContainerView::GetAccessiblePanes checks child visibility, so callers
  must filter hidden parents. MaybeUpdateSplitView suppresses the native pair
  in tab fullscreen; MaybeUpdateDevtools iterates native split membership.
  These need coordinated future adaptation, not an active/inactive facade.
- **Additional dynamic-lifetime trap:** ContentsBorderController (`:26`) creates
  per-container attachment subscriptions once from GetContentsContainerViews.
  A dynamic host needs explicit add/remove lifetime integration. ScrimViewController
  (`:44`) enumerates the same getter on tab changes and resolves TabInterface.
  Merely constructing capture borders/scrims in new containers does not wire
  their controllers. Foreground protection is still TabStripModel's active/
  native-pair contract (T10), independent of native view visibility.

**Decision:** no BrowserView common-host abstraction or routing change yet.
A narrow, disabled rendering harness is verifiable without pretending that the
above contracts are integrated. No recursive MultiContentsView instantiation.
No ordinary browser production source or native pair behavior changed.

## Implemented checkpoint

New owned `patches/helium/ui/workspaces/views.patch` adds:

- `chrome/browser/ui/views/frame/workspace_contents_view.{h,cc}` in `:ui`.
  `kWorkspaceContentsHost` defaults disabled, checked on construction. There is
  no production caller, command, about:flags entry or BrowserView activation
  path; enabling the feature alone cannot expose a tiling browser mode.
- `Present(model, metrics, bindings)` projects current DIP viewport through T05
  physical-pixel geometry. Callers resolve opaque TabKeys afresh to externally
  owned WebContents; pointers are not retained as identity. Before mutation it
  rejects invalid geometry/scales, missing/null/cross-profile/duplicate displayed
  bindings and DIP-collapsed leaf rectangles. Invalid input preserves existing
  attachments. Hidden bindings are not required.
- Dynamically creates one real ContentsContainerView per displayed leaf, with
  no two-leaf limit; a selected tabbed child may be an entire split subtree.
  Detaches all old contents before creating the next tree-order projection.
  Clear/destruction detach without closing pages. Container lifetime references
  are removed before native view deletion. This bounded harness recreates
  containers per successful presentation; retained controllers/focus are future
  work, not a claimed reconciliation transaction.
- Shared physical edges are rounded back to DIPs, not independent widths;
  viewport edges are pinned to original DIP extents. Parent Views mirroring is
  disabled because geometry already emits physical RTL coordinates. Leaves
  retain native internals. Caller-supplied metrics are policy inputs, not audited
  final browser minimum sizes. Presentation must be repeated on resize/scale
  changes; there is no lifecycle/controller observer yet.
- `workspace_contents_view_browsertest.cc` in `browser_tests`, using
  InProcessBrowserTest, real navigated WebContents, BrowserView, native containers
  and Views layout APIs (not mocks or substitute rectangles). It specifies four
  attached leaves, exact fractional-scale/RTL DIP bounds, invalid-binding
  no-op, selected split-subtree switching (4 → 2 → 1 → 0 leaves), external page
  survival, and unchanged ordinary host attachment. Scoped cleanup removes the
  harness even after an assertion. **Compiled, not executed.**

Headers/dividers reserve geometry only (T07/T08). No active host/primary AX,
foreground, tab lifecycle, native-split exclusion, focus restoration, dialog,
DevTools or fullscreen integration is claimed. The harness suppresses Views
subtree events, not all native input. Caller must not attach contents already
hosted elsewhere and must Clear before transfer; the future controller must
make that precondition enforceable for real tabs.

## Verification and limitations

Starting canonical HEAD `0e15032beaa465160fb89fbe1b73c310d26ca833`;
T HEAD remains `fbd20c49f1c3a8d0057a8714e5c9e98539cd2a21`;
P HEAD remains `2686b7c1cff1daccb5d5fbfa15c63a00fb236814`.
Evidence and backups: `/tmp/i3-t06/` (ephemeral).

1. **Preflight replay passed:** merged and canonical each 347 patches / 1,632
   files, fuzz=0, zero differences. C/T series matched; merged/applied order
   matched. Saved C/T/P status, patches/.pc archive and both owned GN files.
   Empty unrefreshed delta at geometry top before creating views.patch.
2. **Final production build passed:** GN generated 32,685 targets / 4,981 files;
   actual `workspace_contents_view.o` compiled with Chromium flags, seven siso
   steps. Container mapping is the documented `/repo`; no args.gn changes:

   ```bash
   docker run --rm --network none --read-only --tmpfs /tmp \
     --user "$(id -u):$(id -g)" -e HOME=/tmp -e SCCACHE_DISABLE=1 \
     -v "$P:/repo:rw" -w /repo/build/src --entrypoint bash \
     chromium-builder:trixie-slim -c '
       buildtools/linux64/gn gen out/Default &&
       third_party/siso/cipd/siso ninja -C out/Default -local_jobs 2 \
         obj/chrome/browser/ui/ui/workspace_contents_view.o'
   ```

3. **Native test object compiled via non-destructive targeted path.** Normal
   siso request for `obj/chrome/test/browser_tests/workspace_contents_view_browsertest.o`
   fails scheduling through browser_tests.inputdeps → extension feature generator
   → missing `chrome/test/data/extensions/extension_api_unittest/api_features.json`.
   Same preexisting blocker as unit_tests; no fixture synthesized or downloaded.
   `siso ninja -t commands` advises `siso query commands`. Used:

   ```text
   third_party/siso/cipd/siso query commands -C out/Default \
     obj/chrome/test/browser_tests/workspace_contents_view_browsertest.o
   ```

   Selected the single command containing ` -c ../../chrome/browser/ui/views/frame/workspace_contents_view_browsertest.cc `
   from its output. Kept generated flags, includes, modules, compiler and sysroot;
   changed only the output prefix from `obj/chrome/test/browser_tests/workspace_contents_view_browsertest`
   to `/e/workspace_contents_view_browsertest`. Ran in the same container at
   `/repo/build/src/out/Default`, mounting `/tmp/i3-t06:/e` and disabling sccache.
   Final compilation exited 0 with no diagnostics, producing a 275,712-byte
   object. Script: `/tmp/i3-t06/compile-test.sh`; final log `verified-build.log`.
   This does NOT mark the scheduled test target built or bypass its dependency
   graph for a runnable binary. Initial compilation caught missing Views metadata
   and obsolete Browser::profile usage; fixed with metadata and GetProfile.
4. **Runtime integration genuinely blocked:** no `out/Default/browser_tests`
   exists and its input dependency is missing. Linking/executing a browser test
   requires that real fixture/build dependency to be restored through a separately
   approved source-resource workflow. No hand-reconstructed browser link, stale
   helium launch, or pure test is offered as native acceptance. No browser test
   or interactive browser launch ran. The intended runtime filter is
   `WorkspaceContentsViewBrowserTest.LiveLeavesAndSelectedSplitSubtree`.
5. **Pure regression only:** reran unchanged T05 ASan/UBSan executable
   `/tmp/i3-t05/workspace_tests`; **52/52 passed**, no findings. Model/geometry
   source and patches remain unchanged. This is NOT native-host evidence.
6. **Final replay passed:** merged and canonical each **348 patches / 1,635
   files**, zero differences. Five owned source-file hashes match after narrow
   pop/push and after final compilation. Empty quilt delta and git diff --check.

## Patch provenance and preservation

Created views.patch after geometry.patch, before all nine platform entries.
Registered source files before creation; edited S first. Used container quilt
refresh with `-p ab --no-timestamps --no-index --strip-trailing-whitespace`,
verified empty diff, saved hashes, popped/pushed only owned patch, verified hashes,
then `push -a` without force or refresh. Read generated patch; later review only
adjusted the harness input-isolation documentation and GN source ordering.
Scratch unmerge routed views.patch into generic output; asserted generic series
was prior C/T series plus exactly its line, platform series equaled series.orig.
Copied only views.patch E → T/C and appended only its approved series entry.

P/T/C patch SHA256:
`ffb0d711491adabbee09bf32bfcc6340f1c5aed7cab090b0e3ccb38483aead5d`.

All **694 preexisting live patch/metadata files** except intentionally changed
series.merged match the snapshot. All **339 C/T generic patch pairs** match.
C/T series and merged/applied order match. Deleted AGENTS, dangling CLAUDE and
build-local.log preserved (recorded SHA256 unchanged). T remains intentionally
dirty with model/geometry/views and series; no setup/reset/pull/broad sync or
unrelated patch refresh. No commit skill available; explicit task-path staging
and CONTRIBUTING scope-first commit convention used. Plan/scout remain untracked
and unstaged; T06 checkbox is unchanged.

## Remaining T06 acceptance / handoff

- Run the native test after restoring the existing test resource legitimately;
  fix runtime failures before checking T06. Add default-gate/ordinary native-split
  regression coverage and further native invalid/resize/lifetime cases as needed.
- ISC-2/3 native hosting and >2 **live displayed** leaves: test authored and
  compiled, not accepted. ISC-A-1: external ownership enforced by API shape and
  detach implementation, runtime survival assertion not executed. ISC-A-4:
  ordinary browser routing unchanged by inspection; runtime regression pending.
- Browser adapter signatures must address enumerated visible/active/per-page
  hosts **and** dynamic controller subscriptions, primary AX selection, focus
  maps and dialog positioning; merely redirecting active/inactive getters is
  unsafe. Do not integrate this test harness as a production mode.
- T07 owns headers; T09 owns stable tab associations/lifecycle; T10 owns actual
  activation/foreground. Further implementation must preserve the disabled gate
  until those contracts and integrated tests are satisfied.
