# T10 transaction hardening — partial implementation

2026-09-17. Continued from `cf0d980` with `PI_MODEL=gpt-6-astra`.
**T10 remains incomplete; T11 has not started. This is not a runnable-feature
handoff or runtime acceptance.** Both features remain disabled by default,
with no public entrypoint. No tests, browser launch, resource restoration,
downloads, or linked-browser build were performed.

## Implemented

Changes are source-first in the existing `controller.patch`:

- Internal deferred activation computes pending destination geometry from a
  copied/revealed model, rather than passing the old displayed set. It resolves
  every destination leaf and rejects invalid/empty geometry. It does not publish
  the copied tree or foreground set before Chromium activation.
- Removed optimistic singleton foreground publication at startup and selection.
  An installed provider's empty result now means no committed attachments;
  only absence of a provider selects ordinary active/native-split policy.
- Replaced the provider's `Unretained` binding with a separate weak lifetime
  binding. Input cancellation no longer shares that lifetime guard.
- Startup rejects empty/unlaid-out windows before native detachment. Start/stop
  exclude nested enablement during transfer; teardown clears observation and
  policy before native callbacks. Startup failure reverses host transfer.
- Controller presentation snapshots the model, passes actual compositor scale
  and RTL direction, and validates lifetime, generation, host identity, live
  membership and fresh contents across native callbacks. Nested presentation
  is suppressed. A failed/stale presentation fails closed by disabling tracking
  and restoring the ordinary host, rather than retaining a partially changed
  host with stale committed foreground data. This is not a retained-layout
  rollback/retry implementation.
- Host presentation replaces the destruction-unsafe `AutoReset` with weak
  cleanup, checks transaction validity after attachment, border, layout and
  control operations, and invalidates in-flight presentation on Clear. Clear
  removes leaf bookkeeping and border subscriptions before native detachment.
- BrowserView publishes an empty workspace host before detaching native slots,
  allowing teardown to reverse a partial install. Install/remove check lifetime
  and host identity across transfer callbacks.
- Active container/web-view/contents-view/size accessors no longer fall back to
  a detached ordinary slot. Host lookup rejects in-progress presentation and
  uncommitted keys; BrowserView checks actual active contents identity.
- Active-tab BrowserView UI routing waits for a matching committed attachment.
  Both the active-tab observer and successful controller commit try to complete
  it, avoiding dependence on their registration order for this UI work. This
  does not yet unify all tab-strip lifecycle observer ordering.
- Scrims refresh on host commit, including startup and layout-only changes.
  Old primary AX designations are revoked before detach; new primary promotion
  follows projection/scrim commit and requires model-active/Chromium-active
  contents agreement. New containers are demoted before attachment.
- Canceled pre-detach still retains the leaf and association. Updated the stale
  controller browser-test source to enable both required gates and expect
  resolution to survive the pre-detach callback; compiled that source only.

## Verification and provenance

Evidence: `/tmp/t10-transactions/` (ephemeral).

1. Initial independent merged and canonical replays each applied **349 patches
   across 1643 source paths with zero differences**. Snapshot:
   `quilt-patches.tgz`; registered source copies: `sources/`.
2. Verified the full platform top, popped only the platform suffix, and edited
   registered files with `helium/ui/workspaces/controller.patch` on top. All
   touched paths were already registered and snapshotted; no new baseline or
   patch registration was necessary.
3. Refreshed only the owned patch. Owned pop/push reproduced **all 17 registered
   file hashes** (`owned-before.sha256`). Platform suffix reapplied without
   force or refresh; the final BrowserView platform hunk applied at its new
   offset normally.
4. Scratch unmerge generic/platform series matched canonical/`series.orig`.
   Exported only the controller patch through scratch → T → C. P/T/C SHA256:
   `10f088728c8b10f68c5b2c38b220a0f219b2b0bc7717a448afd907a3ab0c3993`.
5. Final independent merged and canonical replays each report **349 / 1643 /
   zero differences** (`final-audit/`). Canonical platform input came from P
   HEAD, not live refreshed platform patches.
6. Final top: `helium/linux/disable-tab-strokes.patch`; applied series equals
   `series.merged`; `quilt diff -z` is empty. Snapshot comparison found only
   `controller.patch` changed in the pre-existing patch tree. All existing
   `.pc` files are unchanged except the expected platform BrowserView backup,
   which now includes this task's generic source changes. The repaired
   controller originals are therefore preserved byte-for-byte.
7. Final compilation on the fully applied/exported source: **nine exit codes
   of 0**, with empty diagnostic logs, in `final-compile/results.log`:

   ```text
   browser_view 0
   browser_window_features 0
   contents_border_controller 0
   tab_strip_model 0
   workspace_contents_view 0
   workspace_controller 0
   workspace_controller_browsertest 0
   workspace_resize_area 0
   workspace_tabbed_header_view 0
   ```

   Production commands are the proven per-source commands from
   `/tmp/t10-astra-ndkz/final-compile`, with unchanged `/evidence` destinations.
   The browser-test object command was extracted from the current generated
   graph using `ninja -t commands`, removing only sccache and redirecting object
   and depfile outputs. Compilation used `chromium-builder:trixie-slim`, P at
   `/repo`, read-only source, networking disabled. No dependency build or GN
   regeneration ran. An initial compile caught use of `View::GetCompositor`;
   corrected to `GetWidget()->GetCompositor`, then recompiled successfully.
8. `git diff --check` reports four single-space unified-diff context lines in
   the quilt-generated patch, not added source whitespace. Unrelated deleted
   AGENTS.md, dangling CLAUDE.md, planning artifacts, build-local.log and dirty
   transport tooling remain untouched. No commit skill was available; used
   CONTRIBUTING's scope-first explicit-path commit workflow.

## Exact remaining T10 obligations

1. **Atomic foreground transaction is not complete.** Internal activation now
   predicts destination geometry, but external strip selection and background
   insertion/removal still lack a shared old/new committed notification
   boundary. `NotifyForegroundTabsWillEnterBackground` still runs before the
   later controller observer for those paths. Complete a model-independent
   preparation/commit contract for all paths, notifications only for leaves
   actually leaving, and reconcile predicted DIP-representable geometry with
   successful native attachment. Do not treat pending intent as foreground.
2. **Finish BrowserView routing.** The active getters and active-tab UI ordering
   were improved, but direct native-host use remains in DevTools, stored-focus
   mapping, accessible panes, modal/accessory/focus paths. Audit nullable active
   getters at all callers and complete routing without disguising an absent
   committed active attachment. Keep deeper T12 accessory policy separate.
3. **Complete lifetime/transaction review.** The major native transfer and
   controller publication seams now have guards and fail-closed rollback.
   This is not proof that every constructor, Views/AX callback or accessory
   reentrancy path is guarded. Startup/empty-bounds rollback compiles but has
   not been exercised. Decide/implement retained-projection rollback or retry
   for interrupted presentation instead of the current conservative disable.
   Provider installation/removal still is not part of obligation 1's atomic
   old/new foreground notification protocol.
4. **Confirmed detach ordering remains coupled to obligation 1.** Pre-detach
   invalidation deliberately preserves cancellation-safe resolution and leaf
   state. Complete confirmed removal/transfer attachment ordering and gesture
   cancellation; do not remove a leaf merely on WillDetach.
5. **Scrim/AX runtime acceptance remains open.** Commit refresh and actual
   active identity checks are implemented, but exactly-one-primary through
   failed/stale/external selection transactions depends on obligations 1–3.
   No accessibility or modal behavior has been exercised.
6. **Renderer/footer/overlay focus remains unimplemented.** Views events are
   still disabled; there is no displayed-key/contents/generation-validated
   renderer-focus activation adapter. Disabling Views subtree events is not
   native focus isolation.
7. **Header/controller mutations and drag transactions remain unimplemented.**
   Headers/dividers still have null callbacks. Implement structural selection,
   cumulative pointer deltas from a frozen baseline, incremental keyboard/AX
   deltas, and generation cancellation. Present still refuses reconciliation
   while a divider is resizing.
8. **The two identified stale controller test contracts are fixed and their
   object compiles.** Future tests must cover the new failure/rollback and
   notification protocol once implemented; none were executed here.

No ISC checkbox is marked passed. T11 native-split boundary exclusion and T12
accessories remain prerequisites to T13/public enablement. The user-requested
T11-complete ping has not been sent. Human testing should be requested only
when a linked, documented runnable feature exists; the future walkthrough in
`T10-post-repair-result.md` remains the handoff checklist, not current results.
