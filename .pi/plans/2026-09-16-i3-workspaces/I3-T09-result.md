# I3-T09 — per-window observation controller and stable tab associations

2026-09-16. **T09 observation-only implementation complete; runtime acceptance
UNVERIFIED.** This is real BrowserWindowFeatures lifetime/discovery plumbing,
not an interactive workspace mode. T10 activation/foreground and T11 exclusion
remain unimplemented. No tests or browser processes ran; no test resources were
restored/downloaded. T06–T08 runtime acceptance remains deferred, not passed.

## Delivered and exposure boundary

New `patches/helium/ui/workspaces/controller.patch`, after views.patch and before
the nine platform patches, owns seven source/GN paths:

- `chrome/browser/ui/helium/workspaces/workspace_controller.{h,cc}`
- `chrome/browser/ui/helium/workspaces/workspace_controller_browsertest.cc`
- `chrome/browser/ui/browser_window/{internal/browser_window_features.cc,public/browser_window_features.h}`
- `chrome/browser/ui/BUILD.gn`, `chrome/test/BUILD.gn`

BrowserWindowFeatures creates one **inactive** controller for each TYPE_NORMAL
window using its real UserDataFactory. `WorkspaceController::From(window)` uses
ScopedUnownedUserData, matching HeliumLayoutStateController. No profile-keyed
storage, prefs, topology persistence, tab owner, or WebContents owner is added.
Normal incognito windows receive independent memory-only controllers; sharing a
Profile does not share model, generators or subscriptions.

`kWorkspaceTabTracking` is disabled by default. Explicit `StartTracking()` also
checks window type and native split membership, returning `kFeatureDisabled`,
`kUnsupportedWindow`, `kNativeSplitPresent` or `kStarted`. The native-split result
means close native splits before retrying; no migration/destruction is attempted.
There is **no production call to StartTracking**, command, flags-page entry or
interactive enable API. Even setting the feature switch does not start tracking
or attach any pages. The production hookup provides lifetime/discovery only.

Start imports existing strip order into one tabbed-root workspace, reveals the
actual active tab and explicitly wraps a singleton. An initially empty strip
gets an empty shell. The controller exposes only a const model and borrowed
fresh tab resolution; it cannot change browser activation or workspace layout
from UI. Native split creation after tracking starts stops/clears the mirror
without touching tabs or the native split. This is defensive observation, **not
T11's model-boundary exclusion**. Do not use it as active tiling exclusion.

## Verified Chromium API / lifetime contract

Read actual applied sources, not proposed API names:

- `components/tabs/public/{tab_interface.h,supports_handles.h,tab_handle_factory.h}`:
  `TabInterface::GetHandle()` returns `tabs::TabHandle`; `handle.Get()` resolves
  on the UI thread and returns null after destruction. Store this value in
  `map<TabKey, TabHandle>`, never a raw WebContents identity or strip index.
  ResolveTab also checks current BrowserWindowInterface, current strip membership,
  tracking state and pre-detach invalidation. Returned TabInterface is borrowed
  for immediate use; call GetContents afresh. No contents cache exists.
- `TabStripModel::DiscardWebContents` and `TabModel::DiscardContents`: the
  pre-discard callback runs before swapping contents; the stable TabModel/handle
  survives and the strip subsequently emits kReplaced. Controller invalidates
  its generation before replacement and retains the exact association/leaf on
  kReplaced. It does not mistake a session-ID mapping update for a new tab.
- `TabStripModelObserver::OnTabStripModelChanged` supplies confirmed inserted,
  removed, replaced, moved and selection changes. This is used for discovering
  new tabs and post-commit activation, instead of trying to subscribe to
  RegisterDidInsert/DidActivate too late on a newly discovered tab.
- `InsertTabAtImpl` establishes opener before `InsertTabAtIndexImpl` commits
  selection and emits insertion. Query `GetIndexOfTab`/`GetOpenerOfTabAt` only
  during that notification; never store indices. A known opener anchors policy
  insertion in its own workspace; otherwise the model uses its active page.
  Foreground is determined by `selection.new_tab`, not multi-selection or strip
  position. Chromium has already interpreted disposition flags at this seam.
  Background insertion leaves active page/workspace unchanged; selection changes
  call RevealTab to select workspace/ancestor paths. No second activation occurs.
- Per-tab RegisterWillDetach marks the key unresolvable immediately, but retains
  its model leaf until confirmed kRemoved. A requested/canceled close does not
  invoke model removal. Confirmed transfer removes source association/subscriptions;
  destination insertion allocates a destination-local key for the live handle.
  Strip reorder does not reorder topology. Pins/groups remain strip metadata.
- `TabStripModel` explicitly **forbids ScopedObservation**: its observer owns
  observation bookkeeping. Use AddObserver and StopObservingAll; ModelDestroyed
  removes the observer before invoking OnTabStripModelDestroyed. StopTracking
  clears per-tab CallbackListSubscriptions, associations, model and detach state.
  Unretained pre-detach/discard callbacks are bounded by those subscriptions.
- BrowserWindowFeatures::TearDownPreBrowserWindowDestruction resets the controller
  in the Init/TYPE_NORMAL teardown section. BrowserWidget's destructor invokes
  that hook before native BrowserView teardown; Browser::~Browser also resets
  features before member tab-strip destruction. Destructor cleanup is idempotent;
  the strip-destruction callback is a defensive alternate-lifetime cleanup path.

TabKey/NodeId/WorkspaceId generators are retained across Stop/Start, preventing
stale keys from aliasing newly imported tabs in the same controller. IDs are
**controller-local**, not globally unique between windows; callers must retain
owner identity. Runtime handles and IDs are not serialization keys.

## T08 / T10 handoff

`generation()` increases for each observed strip change, pre-detach, pre-discard,
start and stop. No view request callbacks, asynchronous tasks or model mutation
commands exist in T09, so no partial drag implementation or weak-callback API is
pretended here. Future input must bind weak controller lifetime plus generation,
revalidate owner/current IDs and lifecycle state, and introduce the real mutation
transaction with T10 activation/foreground reconciliation.

For cumulative pointer resize, retain one baseline model/target/viewport/metrics
for the gesture; recompute every cumulative displacement from that baseline.
Invalidate on external lifecycle/topology/workspace/viewport/scale changes, do
not silently rebase. Keep captured surface identity alive and update committed
AX values; rebuild only after completion. Keyboard and pointer enter the same
ResizeWorkspace mutation boundary. Generation is an invalidation ingredient,
not a completed transaction/drag implementation.

**No WorkspaceContentsView construction, attachment, event enabling or host
routing was added.** T06's harness remains isolated, disabled and display-only.
T10 must address primary AX, real activation, visibility/foreground, stale focus,
title notifications and dynamic host contracts before any real tab attachment.
T11 must block all native split creation/restore/drop paths at the authoritative
boundary. T12 retains fullscreen/dialog/DevTools work. Safe input exposure still
requires those contracts and an explicitly authorized runtime-validation phase.

## Compilation and static evidence (not runtime tests)

Starting C HEAD `e9c632841d120133cb143da2c8bac5a5f8532cde`.
T HEAD `fbd20c49f1c3a8d0057a8714e5c9e98539cd2a21`;
P HEAD `2686b7c1cff1daccb5d5fbfa15c63a00fb236814`.
C/P/T/S paths and container quilt wrapper are in implementation-workflow.md.
Ephemeral commands, source hashes, snapshots and logs: `/tmp/i3-t09/`.

- Read plan, scout, workflow, validation-status, T06/T07/T08 reports, CONTRIBUTING
  and native tab/window implementations. No commit skill available in advertised
  or searched local skills; use explicit task staging and scoped commit message.
- Preflight merged/canonical replay: **348 patches / 1,639 files**, fuzz=0,
  zero differences. C/T series and merged/applied order matched. Saved statuses
  and patches/.pc archive; empty unrefreshed delta at views top before new patch.
- Final production compilation: GN generated **32,685 targets / 4,981 files**;
  siso **8 steps succeeded**, including both changed production translation units:

  ```bash
  docker run --rm --network none --read-only --tmpfs /tmp \
    --user "$(id -u):$(id -g)" -e HOME=/tmp -e SCCACHE_DISABLE=1 \
    -v "$P:/repo:rw" -w /repo/build/src --entrypoint bash \
    chromium-builder:trixie-slim -c '
      buildtools/linux64/gn gen out/Default &&
      third_party/siso/cipd/siso ninja -C out/Default -local_jobs 2 \
        obj/chrome/browser/ui/ui/workspace_controller.o \
        obj/chrome/browser/ui/browser_window/internal/internal/browser_window_features.o'
  ```

- Final native browser-test object: generated-command compilation exited **0**,
  no diagnostics, **312,176 bytes**. Queried `siso query commands -C out/Default
  obj/chrome/test/browser_tests/workspace_controller_browsertest.o`, selected only
  the exact source compiler invocation, retained generated flags/includes/modules/
  sysroot/compiler, changed only object/depfile prefix to `/e/`. Ran in the same
  container in out/Default with `/tmp/i3-t09:/e`. Script `compile-test.sh`, log
  `final-build.log`. No dependency graph changes or runnable-binary link bypass.
- Initial compile caught TabStripModel's forbidden ScopedObservation, missing
  concrete Profile include and namespace-level AddTabTypes flags; corrected using
  actual APIs. An initial production request used a wrong object target; corrected
  to browser_window/internal/internal. An intermediate request while platform
  suffix was popped hit the unpatched Node-version action; reapplying the original
  suffix restored its normal build behavior. No resource repair/download occurred.
- Four browser tests authored and compiled, **NOT RUN**: initial tabbed import/
  order/opener background insertion/foreground selection/replacement identity;
  pre-detach invalidation/confirmed transfer/independent normal and incognito
  controllers/stop-unsubscribe/re-enable IDs/real window teardown; unsupported
  popup/native-split rejection and defensive stop; explicit disabled gate.
  They exercise the tab-strip disposition seam, not end-to-end navigation requests.
- Final **post-build** merged/canonical replay: **349 patches / 1,642 files**,
  fuzz=0, zero differences. All seven source hashes match after owned pop/push,
  platform suffix reapplication and final compile. Source added lines have no
  trailing whitespace. Standard staged diff --check reports only four required
  single-space unified-diff blank context prefixes in the new patch; with
  `git -c core.whitespace=-blank-at-eol diff --cached --check` it passes. Generated
  patch context was not hand-edited to suppress that artifact warning.

No tests (including pure/model), browser launches, interactive actions or full
browser/test executable builds were run. Existing missing test fixture remains
untouched. Compilation is not runtime acceptance, cancellation/restore validation,
foreground safety or ordinary-window regression evidence.

## Patch provenance and preservation

Registered all seven paths with quilt **before edits**. New controller.patch sits
after views.patch, before platform suffix. Edited applied source only; refreshed
only controller.patch with `-p ab --no-timestamps --no-index
--strip-trailing-whitespace`, inspected the generated patch, verified empty delta,
owned pop/push hashes and normal suffix push (no force/refresh). Scratch unmerge
verified generic series equals previous series plus exactly controller.patch and
platform series equals series.orig. Exported only controller.patch E -> T -> C;
appended only its approved C/T series entry.

P/T/C controller.patch SHA256:
`e27dcd991121c6417cf234af1c3fdb6672da05b0481b3f8f5d6aa95d3e18f6f7`.

Of **696** preexisting live patch/metadata files, only series.merged changed.
All **340** C/T generic patch pairs match, and P/T/C controller bytes match.
Existing model/geometry/views patches are untouched. P status path list matches
preflight; T remains intentionally dirty with workspace patches and series.
Deleted AGENTS.md, dangling CLAUDE.md, untracked plan/scout/blocker and
build-local.log were not restored or staged. No setup/reset/pull/broad sync.

## Acceptance status

Implementation supports ISC-1/7 and A-1/A-3: per-window state, confirmed-removal
policy, stable fresh resolution, no second tab owner or persistent indices.
Replacement identity, unsubscribe/teardown and window/incognito independence
have compiled test assertions, **not passed runtime evidence**. ISC-A-4 is
supported by inactive/default-disabled routing inspection only; native ordinary
window regressions remain unverified. ISC-A-2's active-mode exclusion, ISC-6/8
activation/foreground and all interactive workspace behavior remain T10/T11+.
No runtime ISC checkbox or release-readiness claim is made.
