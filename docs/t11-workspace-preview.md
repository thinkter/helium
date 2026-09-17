# T11-only workspace developer preview

**Not production-ready. T12/T13 are not implemented; runtime is unverified.**
Do not use a normal profile. No topology persistence, workspace commands,
shortcut settings, or general enable/disable UI are provided.

## Build

Use the existing `chromium-builder:trixie-slim` image, source and `out/Default`
arguments. The current Dawn Go launcher is a symlink to `/usr/bin/go`; the
image's Go 1.24.4 is too old. The locally installed, static Go 1.26.5 works
without installing or downloading a toolchain. This command mounts it read-only:

```bash
P=/home/ashman/Documents/rubbish/helium-linux
B=$(mktemp -d /tmp/helium-preview-build-XXXXXX)
mkdir -p "$B/home" "$B/go-cache"
set -o pipefail
docker run --rm --network none --user "$(id -u):$(id -g)" \
  -v "$P:/repo:rw" -v "$B:/evidence:rw" \
  -v /usr/lib/go:/opt/host-go:ro \
  -v /usr/lib/go/bin/go:/usr/bin/go:ro \
  -v /home/ashman/go/pkg/mod:/opt/go-modcache:ro \
  -e HOME=/evidence/home -e GOROOT=/opt/host-go \
  -e GOTOOLCHAIN=local -e GOPROXY=off -e GOSUMDB=off \
  -e GOPATH=/evidence/go -e GOMODCACHE=/opt/go-modcache \
  -e GOCACHE=/evidence/go-cache -e SCCACHE_DISABLE=1 \
  -w /repo/build/src --entrypoint bash chromium-builder:trixie-slim -c '
    export PATH=/opt/host-go/bin:$PATH
    third_party/siso/cipd/siso ninja -C out/Default -offline -local_jobs=2 chrome
  ' 2>&1 | tee "$B/build.log"
```

This builds the real browser dependency graph, not scratch objects. Do not
launch unless the full command succeeds. `out/Default/helium` existing on disk
is **not** evidence that it contains this preview. See
[build provenance](../.pi/plans/2026-09-16-i3-workspaces/T11-preview-result.md).
Do not restore test fixtures or bypass missing dependencies to get a build.

## Human launch (only after a successful full build)

```bash
PROFILE=$(mktemp -d /tmp/helium-workspace-preview-XXXXXX)
/home/ashman/Documents/rubbish/helium-linux/build/src/out/Default/helium \
  --user-data-dir="$PROFILE" --no-first-run --no-default-browser-check \
  --helium-workspace-developer-preview \
  --enable-features=WorkspaceTabTracking,WorkspaceContentsHost \
  about:blank
```

Keep the component build directory intact. Do not run as root or disable the
sandbox. Use a **new** `mktemp` directory on every attempt; do not substitute
an existing profile, add startup URLs, or use incognito. Startup requires an
absolute user-data directory whose basename starts `helium-workspace-preview-`,
a newly created normal profile, one `about:blank` tab, both disabled-by-default
features and the explicit preview switch. At most one eligible window is
modified per process, after nonzero normal layout. Closing/stopping never
re-enables it automatically. Other windows remain ordinary.

## Focused manual checklist

Use a wide window initially. Four ordinary Chromium tabs should appear:

```
left warning page | Tabbed[Right single page, Vertical[upper, lower]]
```

- Initially the left page and two nested right panes are visible. Click the
  right header's single-page selector: only that subtree changes. Select its
  nested-split selector: both upper and lower pages return.
- Click/type in each page's input. Global active tab and omnibox must follow
  the actual focused pane. Select a hidden tab in the global strip: its header
  ancestry must reveal it, without closing any page.
- Drag the main vertical divider and the right horizontal divider. Both sides
  resize; header switching retains the nested split and its weights.
- Narrow the window: constrained fallback may show only the active page.
  Widen it again: topology should return.
- Close a visible or hidden tab normally: only the confirmed closed leaf goes
  away. Reorder/pin/group tabs: global metadata must not rearrange the tree.
  Move a tab to a new window: it disappears from the source only on confirmed
  transfer; the destination is ordinary, not automatically tiled. Closing the
  final tab should follow normal window-close behavior.
- Native split creation should be unavailable while tiled. In a new ordinary
  window native splits should still work. A low-level native restore/transfer
  may end tiling before preserving native split data.
- Fullscreen or opening DevTools deliberately **ends** this preview and returns
  to ordinary hosting. Exiting/closing them does not restore the tree. Start a
  fresh disposable profile to try the sample again.

**Avoid dialogs, permissions, find, side panels, zen/accessory UI, capture and
real browsing workflows in this limited preview.** Their geometry/targeting,
existing-modal transfer and accessibility are not accepted or validated. A
sample page visibly warns of these gaps. Canceled-beforeunload and renderer
lifecycle behavior still need later authorized runtime validation. Do not
interpret this checklist as a list of passed tests. Report any crash, wrong
active tab, blank attachment or unexpected ordinary-window change and stop.

After all preview windows exit, the disposable `$PROFILE` can be removed.
Normal tab session restore remains Chromium-owned; tiling topology is never
saved. Remove the switch/features to launch an ordinary browser.
