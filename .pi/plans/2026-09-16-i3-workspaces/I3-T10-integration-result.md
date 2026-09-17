# I3-T10 continuation — retained-host checkpoint, integration INCOMPLETE

2026-09-16. **T10 is still incomplete; do not start T11 on the strength of this
checkpoint. Runtime UNVERIFIED.** This continuation implements retained native
leaf/control reconciliation and per-leaf capture-border subscription lifetime.
It does **not** deliver the requested production foreground/BrowserView/focus
transaction. Those remain T10, not deferred T11/T12 responsibilities. No tests,
browser launches, resource restoration or downloads were performed. There is
no new user entrypoint or request for intermediate human testing.

## Implemented

Only `controller.patch` was refreshed, layering on the existing views sources.
The original model/geometry/views patches and generic/platform series are intact.

- `WorkspaceContentsView` now keys retained native containers by **TabKey**, not
  geometry slot, tab-strip index or WebContents address. Resize, structural
  selection and active-leaf changes that retain the displayed keys do not
  detach/reattach their pages. Replacement changes only the attachment and keeps
  the native container. The binding map remains fresh input to Present, not a
  retained WebContents identity cache or owner.
- Reconciliation validates geometry, bindings and control projections first.
  It detaches **all** outgoing/replaced attachments before attaching any new
  ones, including swaps between existing keys. Only disappearing keys lose
  containers. Current tree order controls enumeration and native child order.
  `GetContainerForTab` resolves a retained host by its stable key; callers must
  still validate current controller ownership/generation before using it.
- Headers persist by container ID; their buttons persist while the same ordered
  immediate-child IDs remain. Selection/title changes refresh AX state, labels
  and borders without destroying button focus identity. Topology changes revoke
  callbacks before rebuilding buttons. Dividers persist by container/index and
  refresh committed geometry/AX values. The structural outline also persists.
- A native per-container capture-border controller is constructed **before first
  attachment** and destroyed **before container deletion**. This is a live
  subscription in the existing host, not an unused provider. Its lifetime lives
  with the retained leaf. BrowserView's ordinary pair controller is unchanged.
- Fixed native `ContentsContainerViewBorderController::OnWebContentsDetached`
  to unsubscribe both capture-change **and capture-location** callbacks and clear
  the old location. Previously the old page could update the border of a reused
  container after replacement with a page lacking a capture helper. This narrow
  bug fix also applies to ordinary native containers.

The host remains a disabled, input-suppressed rendering harness. Control
validation currently constructs temporary unattached control candidates before
updating retained controls; it does not recreate attached native page hosts.
Present explicitly rejects an in-progress divider drag, rather than deleting or
retargeting its captured surface. The controller's baseline/gesture transaction
is still required before enabling input: retained identity alone is not working
production pointer resizing. No primary AX designation, foreground provider or
BrowserView transfer path was added, and none is claimed as integrated.

## Compiled assertions, NOT RUN

`workspace_contents_view_browsertest.cc` now specifies:

- four displayed native pages retain the same containers/dividers across
  selection and viewport changes, with **zero** WebView detach/attach callbacks;
- fresh bindings replacing/swapping contents retain host identity and every old
  attachment detaches before the first new attachment;
- repeated tabbed presentation retains headers and their immediate-child buttons;
- invalid missing bindings leave the current native tree unchanged;
- native capture subscriptions initialize on attachment, leave both subscriptions
  behind on replacement, cannot modify the reused container from the old page,
  resubscribe correctly on reattachment and expire on Clear/destruction;
- external WebContents ownership survives host teardown.

These are compilation evidence only. They do not test actual browser-owned-tab
foreground, command targeting, transfer between browser hosts or native renderer
focus. Existing standalone header/divider assertions also still compile.

## Exact remaining T10 work (original eight-point audit)

1. **Unimplemented:** a Views-independent, actually displayed-tab contract used
   by both `TabStripModel::GetForegroundTabs` and `IsTabInForeground`. It must be
   fed by successful committed native attachments, not merely policy visibility.
2. **Unimplemented:** correct `SetSelection` / insertion / close notification
   ordering. `NotifyForegroundTabsWillEnterBackground` must not notify leaves
   that remain displayed when only the active tab changes. Outgoing visibility
   and active-tab deactivation require one coordinated transaction.
3. **Unimplemented:** BrowserView active/per-page/all-visible getters,
   OnActiveTabChanged, OnTabDetached, native pair direct attachment paths,
   stored-focus mapping and F6 traversal must switch together with explicit
   detach-before-attach transfer. Current ordinary routing remains unchanged;
   the workspace host still has no production constructor/caller. Its external
   no-double-hosting precondition is not yet enforced by a browser adapter.
4. **Partly implemented:** workspace leaf capture-border subscriptions now have
   dynamic lifetime, and native detach clears both capture subscriptions. The
   production adapter must retain this single ownership arrangement (do not
   additionally register the same leaf in BrowserView's constructor-time pair
   controller), then connect any further dynamic host consumers.
5. **Unimplemented:** scrim refresh during each committed host reconciliation
   and correct per-page resolution. Re-enumeration alone is insufficient.
6. **Unimplemented:** page/footer/actor/reading-overlay focus subscriptions and
   class mapping; guarded renderer-focus requests validating fresh tab handle,
   contents, actual displayed leaf and presentation generation before the
   already-existing deferred Chromium activation backend is called.
7. **Unimplemented:** demote/detach native primary AX on transfer and designate
   exactly the actual active workspace page as primary. No leaf is made primary
   in this checkpoint.
8. **Partly implemented:** retained leaf/header/button/divider/outline identity
   replaces recreate-all attached hosting. Still required: controller-driven
   attachment/visibility transaction, stale request invalidation, committed
   getter coherence, and cumulative drag baseline plus cancellation/revalidation
   for topology/workspace/lifecycle/viewport/scale changes. Input remains disabled.

No build blocker prevents further implementation. This is a bounded checkpoint,
not a claim that the whole assigned continuation was completed. T11 exclusion,
T12 accessories and T13 exposure remain untouched and are not excuses to omit
these remaining T10 obligations. No ISC runtime checkbox is newly satisfied.
ISC-A-1/A-3 remain structurally preserved (external ownership and stable keys).
ISC-A-4 is supported only by static ordinary-path isolation plus the narrow
capture-detach fix; runtime regression evidence remains absent.

## Verification and provenance

Starting C HEAD: `bed1520`. T and P retain the revisions recorded in T09/T10.
Read plan, workflow, validation-status, scout, CONTRIBUTING, T06–T10 results and
actual local host/controller/capture implementations before edits. Evidence is
in `/tmp/i3-t10-integration/` (ephemeral); reused the reviewed container-quilt,
full-replay and scratch-unmerge scripts from `/tmp/i3-t10/`.

- Preflight merged/canonical full replays: **349 patches / 1,642 files**, fuzz=0,
  zero differences. Saved statuses and patches/.pc archive before source edits.
- Verified previously unpatched `contents_border_controller.cc` byte-for-byte
  against the corresponding member of the existing local
  `chromium-153.0.8010.36-lite.tar.xz` source archive. No baseline/resources were
  restored into S. Registered all six newly layered paths before editing.
- Source-first edits in S; only controller.patch refreshed with `-p ab
  --no-timestamps --no-index --strip-trailing-whitespace`. Empty unrefreshed diff.
  All **13 owned source hashes** match after narrow owned pop/push, original
  platform suffix reapplication and final compilation. No forced quilt actions.
- Reviewed generated patch/source delta. Scratch unmerge verified unchanged
  generic and original platform series; exported only controller.patch E → T → C.
  All **340 C/T generic patch pairs** match. Of **697 archived live patch and
  metadata files**, only controller.patch changed. P/T status path lists match
  preflight; dirty tooling remains intact.
- Final production object build succeeded in **10 siso steps**, including all
  four affected translation units, with the documented network-disabled
  container, SCCACHE_DISABLE=1 and actual Chromium compiler flags:

  ```text
  third_party/siso/cipd/siso ninja -C out/Default -local_jobs 2 \
    obj/chrome/browser/ui/ui/workspace_contents_view.o \
    obj/chrome/browser/ui/ui/workspace_tabbed_header_view.o \
    obj/chrome/browser/ui/ui/workspace_resize_area.o \
    obj/chrome/browser/ui/ui/contents_border_controller.o
  ```

- Updated browser-test object compiled successfully with no diagnostics:
  **839,976 bytes**. Obtained its exact command from `siso query commands`,
  preserving flags/includes/modules/compiler/sysroot and changing only the
  object/depfile output prefix to `/e/workspace_contents_view_browsertest`.
  No native test target linked, graph changed, fixture synthesized or executable
  run. Script `compile-test.sh`; final production/test log `final-build.log`.
- Final **post-build** merged/canonical full replays: **349 patches / 1,643
  files**, fuzz=0, zero differences. Logs `post-build-replay.log` and
  `source-sha256`. Source lines have no trailing whitespace. Standard Git patch
  diff checking reports only unified-diff blank context prefixes;
  `git -c core.whitespace=-blank-at-eol diff --check` passes.

P/T/C controller.patch SHA256:
`3392301670b670793b086f66484af30507a94763c7da38c450651fb137fab657`.

No commit skill exists in the advertised/local skill directories; use the
CONTRIBUTING scope-first convention and explicit task-owned staging. Deleted
AGENTS.md, dangling CLAUDE.md, build-local.log, untracked scout/plan/blocker and
unrelated platform state were not restored or staged. No setup/reset/pull/broad
sync. Local untracked plan records this partial status and remains unstaged.
