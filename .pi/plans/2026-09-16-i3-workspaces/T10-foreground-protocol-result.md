# T10 foreground / confirmed-detach protocol

2026-09-17, from `4cb8c25`, `PI_MODEL=gpt-6-astra`. Focused implementation of
obligations 1 and 4 from `T10-transactions-result.md`; **not completion of T10**.
Both gates remain default-off, with no public mode. T11 has not started and the
requested T11-complete notification has not been sent.

## Implemented contract

- `TabStripModel::OnChange` calls a Views-independent lifecycle callback before
  its active-tab processing and ordinary observers, including Browser. The
  controller no longer reconciles in its later `OnTabStripModelChanged`
  observer. Internal deferred activation, external selection, foreground and
  background insertion, replacement, and confirmed removal share this seam.
- Activation sends `WillDeactivate` to the outgoing **active** tab before
  changing selection. It does not infer hiding from deactivation. The host's
  preparation callback sends `WillBecomeHidden` only to committed attachments
  absent from the validated destination, immediately before native changes.
  Retained split-tree leaves therefore do not receive hide notifications just
  because a different leaf becomes active.
- Removed pending selection prediction and its foreground-adjacent state.
  Preparation now uses the host's actual geometry and shared-edge physical
  pixel-to-DIP conversion, after all page bindings and nonempty DIP rectangles
  validate. Prediction and native projection cannot disagree through two
  separate geometry calculations. Invalid geometry retains the existing
  conservative reverse-transfer/disable policy, not a speculative publication.
- Foreground publication follows successful native commit. The provider also
  checks each committed key against its actual current attachment, excluding
  already-detached leaves during callbacks. Both `GetForegroundTabs` and
  `IsTabInForeground` honor an installed provider's empty result; neither
  substitutes the newly selected tab for an uncommitted attachment.
- Enablement imports the active tab into a tabbed root and installs empty
  policy until actual attachment commits. Disablement prepares the outgoing
  difference against ordinary active/native-split policy, without deactivating
  a retained active page, and keeps the provider installed and empty during
  reverse transfer. Ordinary policy resumes only after that transfer returns.
- `WillDetach(kDelete)` remains a request: generation invalidation only, no
  association, tree, or attachment removal and no speculative hide/deactivate.
  The confirmed hook in `DetachTabImpl` runs after beforeunload acceptance and
  before ownership leaves the source model. Collection transfers use the same
  hook before removal. Active deactivation and actual outgoing hiding precede
  native detach; `kRemoved` repairs the tree at the pre-Browser seam.
- Confirmed native retirement invalidates presentation and removes capture
  subscriptions, primary designation, leaf bookkeeping and obsolete controls.
  A controller presentation guard prevents native detach/layout callbacks from
  reattaching the leaf while its confirmed `kRemoved` is still pending.
  Generation and deferred-input invalidation reject obsolete requests.
- Lifecycle callbacks use weak controller lifetime; strip callbacks are copied
  across policy replacement. Hide iteration uses tab handles, strip weak
  lifetime, a policy epoch and callback-stack notification tracking. Nested
  disable does not re-notify an already-delivered outgoing leaf. Stop is
  reentrancy guarded, suppresses presentation while stopping, and an already
  stopped destructor does not look up a potentially destroyed tab strip.

## Regression assertions authored (compiled, NOT executed)

In `workspace_controller_browsertest.cc`:

1. `ForegroundPreparationPrecedesNativeCommit`: background tabbed insertion
   emits no deactivate/hide; internal and external selection order is
   deactivate -> hide with old attachment still present -> activate with new
   committed attachment; pending active intent is not foreground.
2. `ConfirmedTransferRetiresSourceAttachment`: request preserves resolution and
   attachment, confirmed deactivate/hide precede retirement, native detach
   cannot re-present through a layout callback, removed association is gone,
   and disabling does not deactivate the retained active page.
3. `CanceledBeforeUnloadKeepsCommittedAttachment`: real beforeunload request
   and dialog cancellation retain leaf, active selection, attachment and
   foreground membership without hide/deactivate notifications. Uses a data
   URL, not restored/downloaded fixture resources.
4. `StopDuringPreparationInvalidatesNativeCommit`: synchronous stop from hide
   invalidates presentation, sends one hide, removes the workspace host and
   restores ordinary foreground policy.

These are source assertions, not evidence that browser scenarios passed.

## Provenance and compilation

Ephemeral evidence: `/tmp/t10-foreground/`.

- Initial independent merged and canonical full replays: **349 patches / 1643
  paths / zero differences**. Saved patch/quilt archive and all 17 registered
  source files before edits. All seven edited paths were already registered;
  no original-file baseline or registration changes were required.
- Source-first edits with `helium/ui/workspaces/controller.patch` on top.
  Refreshed only that patch. Owned pop/push reproduced **all 17 source hashes**
  (`owned-before.sha256`). Platform suffix reapplied normally without force or
  refresh; final top is `helium/linux/disable-tab-strokes.patch`.
- Scratch unmerge (`review-export/`) verified generic series against C/T and
  platform series against `series.orig`. Copied only controller.patch via
  scratch -> T -> C. Final P/T/C SHA256:
  `313a89e74c02a191216bc15ff640e1bbc104c483455cd8f827cc26c145b182f9`.
- **Final full merged and canonical replays**, after the last safety review:
  **349 / 1643 / zero differences** (`review-audit/`). Canonical platform input
  was P HEAD, not refreshed live platform files.
- Archive comparison: only `patches/helium/ui/workspaces/controller.patch`
  changed; **all archived .pc file contents are unchanged**. Applied order
  equals `series.merged`; `quilt diff -z` is empty. Platform and transport Git
  status listings match their starting listings (owned transport patch content
  intentionally updated). No manual .pc changes.
- Final compilation on fully applied/exported source: **nine exit codes 0**,
  empty diagnostic logs (`final-compile/results.log`):

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

  Reused the proven generated per-source commands from
  `/tmp/t10-transactions/final-compile` (production provenance from
  `/tmp/t10-astra-ndkz`), retaining `/evidence` output destinations. Container:
  `chromium-builder:trixie-slim`, P mounted read-only at `/repo`, networking
  disabled, working directory `/repo/build/src/out/Default`. No dependency
  builds or GN regeneration. Initial compilation identified a missing complete
  `SplitTabData` include; added its real header and recompiled successfully.
- No tests, browser execution, fixture restoration, downloads or linked browser
  build. `git diff --check` reports six single-space unified-diff context lines
  in the generated quilt patch, not added source whitespace. Unrelated deleted
  AGENTS.md, dangling CLAUDE.md, untracked planning files/build-local.log and
  existing dirty tooling/platform state were preserved. No commit skill is
  available; followed CONTRIBUTING's scope-first, explicit-path workflow.

## Remaining T10 / acceptance

- Obligation 2: complete the BrowserView native-host/focus/accessory routing and
  nullable getter audit. This slice does not complete DevTools or modal policy.
- Obligation 3: broader native/Views/AX reentrancy review and retained-projection
  rollback/retry remain. Failed presentation still conservatively disables
  tracking/restores the ordinary host rather than retaining a previous tiling
  projection. This is not a proof covering every native callback or allocation
  failure. Multi-leaf failure/rollback and constrained-DPI scenarios still need
  broader assertions and authorized runtime acceptance.
- Obligation 5: scrim/AX/modal runtime acceptance remains unverified.
- Obligations 6/7: renderer/footer/overlay focus activation and header/divider
  controller mutations, frozen drag baselines and full gesture transactions
  remain for the next slices. This change retires obsolete controls on confirmed
  detach; it does not implement interactive resize/drag behavior.
- T11 native-split boundary exclusion and full groups/pins/transfer coverage,
  then T12 accessories, remain prerequisites to public enablement.

No ISC checkbox is marked passed. This provides compiled/static evidence for
ISC-6/7/8/11/12 and ordinary-policy isolation, not runtime acceptance or a
runnable-feature handoff.
