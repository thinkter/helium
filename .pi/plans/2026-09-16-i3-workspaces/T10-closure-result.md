# T10 finite closure

2026-09-17, starting `25844e9`. **T10 IMPLEMENTATION COMPLETE; runtime UNVERIFIED.**

Fixed the review's P1 in BrowserView::GetWebContentsModalDialogHostFor:
page-specific lookup now uses the workspace-aware accessor, not the detached
ordinary host. WorkspaceContentsView registers its container before calling
SetWebContents; GetDisplayedContainers exposes these attachments during Present,
unlike the deliberately commit-only GetContainerForTab. Thus synchronous
ContentsWebView::SetWebContents -> UpdateDialogHost -> delegate lookup resolves
the existing dialog's own page while attachment is still in progress.

For an unmatched page, ordinary active-host fallback is preserved. During a
workspace transition with no committed active attachment the result is nullptr.
NativeWebContentsModalDialogManagerViews::HostChanged explicitly accepts null,
unsubscribes the prior host, and only reparents/repositions when a host exists.
The manager's UpdateDialogHost also passes null when its delegate is absent.
No detached ordinary page is substituted as a fake workspace attachment.

## Evidence

- `/tmp/t10-closure`: preflight and final FULL canonical and merged replay each
  **349 patches / 1643 paths / zero differences**.
- Snapshot of patch tree/quilt metadata and BrowserView before editing; verified
  controller.patch top and existing registration. Only that source was edited.
- Narrow quilt refresh; owned pop/push reproduced all **19 registered hashes**;
  normal platform push restored disable-tab-strokes.patch top. Empty quilt delta;
  applied order equals merged series.
- Scratch unmerge verified generic and platform series. Only controller.patch
  exported scratch -> transport -> canonical. SHA256:
  `76502378567ec774b17269d06718868a827e5aa0e87bd8f779015403e9ff34d8`.
- BrowserView object compiled **exit 0**, empty diagnostics, using byte-identical
  `/tmp/t10-pane-input/final-compile/browser_view.command`. Existing container,
  network disabled, source read-only at /repo, outputs /evidence. No graph edits.
- No regression source added in this narrow closure. Static call-chain evidence
  above is not execution of an existing-modal scenario.

The reviewer-approved conservative fail-closed reverse transfer remains; no
retained rollback project was added. Modal positioning, fullscreen, DevTools
and broader accessory policy remain T12. Both feature gates remain OFF by
default and no public enable commands exist. No tests, browser, resources or
downloads were run. No ISC/runtime acceptance is claimed. T11 may now proceed.
Unrelated canonical/platform/transport dirty state was preserved.
