# I3-T10 — bounded activation checkpoint; integration INCOMPLETE

2026-09-16. **T10 remains incomplete, not checked off. Runtime UNVERIFIED.**
This checkpoint implements actual Chromium tab activation with stale/reentrant
request guards. It does **not** implement the displayed-tab contract, coordinated
visibility transaction, BrowserView host routing, or interactive tiling. Those
remain T10 work, not work silently reassigned to T11/T12. No tests or browser
processes ran, and no resources were restored/downloaded.

## Implemented contract

Only `controller.patch` changes, in the existing registered
`workspace_controller.{h,cc}` and `workspace_controller_browsertest.cc` paths.

- `RequestActivateTab(TabKey, expected_generation)` resolves a live stable tab
  association in this window, rejects obsolete/invalid requests and native
  splits, and queues actual `TabStripModel::ActivateTab`. Its boolean means
  **accepted for dispatch**, not completed activation.
- Dispatch is **non-nestable**: even a nested modal event loop cannot execute
  activation while the originating tab-strip mutation/observer stack is active.
  The latest accepted request supersedes older queued requests via weak pointers.
- Dispatch rechecks generation, native splits and fresh window membership.
  Selection, insertion/removal, pre-detach, replacement/discard and Stop/Start
  invalidate obsolete input. StopTracking explicitly cancels pending weak tasks;
  destruction cannot leave a callable raw controller pointer in the task queue.
- No optimistic tree-only activation occurs. Chromium commits selection and its
  normal browser/toolbar/command-target updates; the existing confirmed strip
  observer invokes RevealTab to select the workspace/ancestor path. No controller
  member is accessed after calling ActivateTab (synchronous callbacks may destroy
  the window). No WebContents pointer is retained by the request.
- This is an explicit activation backend, **not a renderer-focus entrypoint**.
  A future focus adapter must additionally validate the currently displayed host,
  leaf, contents association and presentation generation before requesting it.
  No production caller, command, flag entry, mutable model exposure or host
  construction was added. StartTracking still only starts observation.

The default-disabled tracking and contents-host features remain disabled.
Enabling the feature switches alone still neither starts tracking nor attaches
real tabs. The isolated host remains noninteractive/recreate-all, with no real
browser-owned tab attachments. No page can become double-hosted through this
checkpoint because it adds no attachment or ownership-transfer path.

## Authored assertions (compiled, NOT RUN)

Two additional native browser tests specify:

- deferred actual strip activation, latest-request wins, unchanged activation
  before dispatch, selected tabbed ancestor after confirmation, stale/invalid
  request rejection and external-selection cancellation;
- request issuance inside WillDeactivate, a nested event loop allowing ordinary
  nested tasks, no recursive strip activation, invalidation by the eventual
  selection notification, replacement/discard, detach and Stop/Start.

These assert real strip state, not only a painted border. They do not constitute
runtime evidence, and do not test multi-leaf foreground or actual visibility.

## Refreshed integration audit / next T10 handoff

Read plan, scout, CONTRIBUTING, workflow, validation-status and T06–T09 reports
fully, and reviewed the actual applied activation/host source. Paths below are
relative to S in implementation-workflow.md. Findings are **not implemented**:

1. `tabs/tab_strip_model.cc:1353,1601`: both GetForegroundTabs and
   IsTabInForeground need the Views-independent displayed-tab contract. Do not
   derive this set from the policy tree alone: constrained geometry, fullscreen,
   lifecycle invalidation and successful attachments determine the actual set.
2. **Additional sequencing obligation:** `SetSelection` (~4190) calls
   NotifyForegroundTabsWillEnterBackground (~6111) outside native same-split
   activation. That helper iterates **GetForegroundTabs** and sends
   WillBecomeHidden to every returned tab. Merely extending the getter would
   incorrectly notify still-displayed panes on every tiling activation. Separate
   actual outgoing visibility notifications from active-tab deactivation within
   the reconciliation transaction, and audit its other insertion/close callers.
   Do not install an otherwise-unused provider and call that active integration.
3. `views/frame/browser_view.cc:2591–2765`: OnActiveTabChanged obtains the active
   WebView before dispatching several direct MultiContentsView detach/reattach,
   split-update and focus-restoration paths. Overriding only the active getter
   would leave a second native attachment path. Coordinate this whole branch
   with host transfer, per-page/visible enumeration, OnTabDetached and stored
   focus. Ordinary inactive windows must retain the current branch unchanged.
4. `views/frame/contents_border_controller.cc`: constructor subscribes only once
   to the containers returned at construction. Dynamic containers require explicit
   add/remove subscriptions **before attachment and before deletion**. Each
   per-container controller retains tab-capture change and location subscriptions;
   OnWebContentsDetached currently clears only the change subscription. A dynamic
   transfer adapter must address both and not retain a callback to a retired view.
5. `views/frame/scrim_view_controller.cc`: updates on active selection, blocked
   changes and split changes, not new workspace projections. Re-enumeration alone
   does not initialize a newly attached container; call the update as part of
   successful host reconciliation, with correct per-page resolution.
6. `views/frame/multi_contents_view.cc:135–170`: page, NTP footer, actor overlay
   and reading overlay each have focus subscriptions and a class-name mapping.
   A dynamic host needs their lifetime management, not only page clicks.
7. `multi_contents_view.cc:339`: primary AX follows active native slot. On
   ownership transfer, demote/detach native slots before attaching workspace
   pages, and make exactly the real active workspace page primary. The isolated
   host currently makes none primary; do not mark all leaves primary.
8. T06 Present destroys/recreates all containers and does not retain live drag
   surfaces. Replace it with retained-container reconciliation before production
   focus/resize routing. Invalidate stale requests during attachment and detach
   old hosts before any new attachment. BrowserView's active/per-page/visible
   getters and F6 traversal must agree with committed attachments.

Next bounded checkpoint should implement the displayed-tab/visibility boundary
and retained host transaction together with notification ordering, then route
BrowserView through it, wire dynamic subscriptions/primary AX and author native
integration assertions. Explicit enablement must stay inaccessible until T11's
model-boundary native-split exclusion and T12's required accessory paths exist.
T11 exclusion and T12 fullscreen/dialog/DevTools/accessory work remain pending;
this checkpoint provides no substitute for either. Tests being prohibited is
**not** a reason to defer implementation; the remaining work needs continuation.

## Compilation and provenance

Starting C HEAD `cae5df5`; T HEAD remains
`fbd20c49f1c3a8d0057a8714e5c9e98539cd2a21`, P HEAD remains
`2686b7c1cff1daccb5d5fbfa15c63a00fb236814`. Evidence/snapshots in `/tmp/i3-t10/`.
No commit skill was available; use CONTRIBUTING's scope-first commit convention.

- Preflight and final **post-build** full merged/canonical replays each passed:
  **349 patches, 1,642 files, fuzz=0, zero differences**.
- Saved statuses and patches/.pc archive. Popped only platform suffix to
  controller.patch. All edited files were already registered by T09; no new
  existing-file seam or original-source verification was required.
- Source-first edits; refreshed only controller.patch with `-p ab
  --no-timestamps --no-index --strip-trailing-whitespace`. Reviewed generated
  delta; empty unrefreshed quilt diff. All seven owned file hashes match after
  owned pop/push, platform suffix reapplication and final compilation.
- Scratch unmerge verified unchanged generic and original platform series;
  exported only controller.patch E -> T -> C. All **340** C/T generic patch pairs
  match. Of **697** archived live patch/metadata files, only controller.patch
  changed. P/T status path lists unchanged; existing dirty tooling preserved.
- Final production object compiled successfully with actual Chromium flags:
  **7 siso steps**, including workspace_controller.o. Command in the documented
  network-disabled container at `/repo/build/src`, SCCACHE_DISABLE=1:

  ```text
  third_party/siso/cipd/siso ninja -C out/Default -local_jobs 2 \
    obj/chrome/browser/ui/ui/workspace_controller.o
  ```

- Queried `siso query commands -C out/Default
  obj/chrome/test/browser_tests/workspace_controller_browsertest.o`; selected
  only the exact source compiler command, retained generated includes/modules/
  flags/compiler/sysroot and changed only output/depfile prefix to `/e/`.
  Executed in out/Default with `/tmp/i3-t10:/e`. Final test object compiled with
  no diagnostics, **390,936 bytes**. No test binary linked/executed, graph bypass,
  source fixture synthesis, download or resource restoration.
- Final log `final-build.log`; hash list `source-sha256`; replay output
  `post-build-replay.log`. `git diff --check` on the owned patch passes.

P/T/C controller.patch SHA256:
`6efb7d5fda6a28580898809ad733ae3f8921caa3f175e99db2c76e85ea14e578`.

Deleted AGENTS.md, dangling CLAUDE.md, build-local.log, untracked scout/plan/blocker
and unrelated platform state were not restored or staged. No setup/reset/pull or
broad synchronization. Local untracked plan records partial T10 implementation;
it remains unchecked and unstaged.

## Acceptance status

ISC-6/12 gain an actual activation backend and compiled confirmation assertions,
not accepted runtime results or complete tiling interaction. ISC-8, coordinated
foreground/visibility, primary AX, host transfer, renderer focus, and >2 displayed
pages remain **unimplemented in T10**. ISC-A-1/A-3 remain structurally preserved
(no new owner or persistent index). ISC-A-4 is supported only by unchanged native
routing/static review; ordinary-window runtime regressions remain UNVERIFIED.
No task-completion, runtime ISC, active-tiling or release-readiness claim is made.
