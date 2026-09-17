# I3-T01 — canonical patch workflow

Date: 2026-09-16. **Accepted preflight; T02 may proceed using the gates below.**
No feature/source edits, quilt push/pop/refresh on the live tree, setup, reset,
pull, substitutions, build, or browser tests were performed. CONTRIBUTING.md,
the approved plan, scout context, and the actual tooling were read completely.

## Authority and verified starting state

Use these host paths throughout:

```bash
C=/home/ashman/Documents/rubbish/helium
P=/home/ashman/Documents/rubbish/helium-linux
T=$P/helium-chromium
S=$P/build/src
```

**C is canonical.** T is a separate clean tooling checkout, not a symlink to C.
P/patches is a live merged working stack, not a canonical export. Changes must
travel from S through narrowly refreshed owned patches, a scratch unmerge,
T, then C. Never synchronize entire patch directories between real checkouts.

| Location | Revision / state at preflight |
| --- | --- |
| C | `fbd20c49f1c3a8d0057a8714e5c9e98539cd2a21`; deleted `AGENTS.md`, untracked `.pi/`, untracked `build-local.log` |
| T | same HEAD; `git status --short` empty |
| P | `2686b7c1cff1daccb5d5fbfa15c63a00fb236814`; merged stack already present |
| S | no `.git`; Chromium `153.0.8010.36`; quilt database version 2 |

C's dangling `CLAUDE.md -> AGENTS.md` is untouched. C/build-local.log SHA256:
`d41f6e270fa558c3690f4b0062b4e36cfd677867c09ead408f003f2d8cee700a`.
Do not stage, restore, remove, or otherwise repair any of these unrelated files.

P has nine modified tracked patches: all six `helium/linux/` entries and all
three `ungoogled-chromium/portablelinux/` entries below. `patches/series` is
removed because the stack is merged. Untracked files include copied generic
patch groups, patch `~` backups, `series.merged`, `series.orig`,
`series.prepend`, `.dev-home/`, and `.helium.pid`. Preserve them.

### Order and content evidence

- C and T: 336 patches, identical series and byte-identical listed patches.
- Merged/applied: 345 patches; `.pc/applied-patches` equals
  P/patches/series.merged byte for byte. Its first 336 entries equal canonical
  order; `series.prepend` equals C/patches/series byte for byte.
- Last generic patch: `helium/ui/exclusive-access-bubble.patch`.
- Platform suffix, in order:
  1. `ungoogled-chromium/portablelinux/drop-nodejs-version-check.patch`
  2. `ungoogled-chromium/portablelinux/use-oauth2-client-switches-as-default.patch`
  3. `ungoogled-chromium/portablelinux/fix-compiling-on-arm64.patch`
  4. `helium/linux/change-chromium-branding.patch`
  5. `helium/linux/rename-chrome-binary.patch`
  6. `helium/linux/use-default-theme.patch`
  7. `helium/linux/add-middle-click-paste-flag.patch`
  8. `helium/linux/add-error-for-missing-desktop-file.patch`
  9. `helium/linux/disable-tab-strokes.patch` (current top).
- All 336 merged generic patch files differ bytewise from canonical files.
  The inspected differences include `Index:` lines and `src.orig/` / `src/`
  headers instead of `a/` / `b/`. **Do not export this refresh churn.**
- Stronger semantic check: reconstructed the earliest quilt backup of each
  touched file into scratch, then applied all 345 merged patches using GNU
  `patch --batch --forward --fuzz=0 -p1`. Result: **345 successes, 1,626
  files checked, zero differences against live S**.
- Repeated reconstruction using C's 336 original patches and P HEAD's nine
  original platform patches: **345 successes, no offsets, zero differences
  across the same 1,626 files**. Thus merged refresh differences do not change
  final source contents on this patch footprint.
- New `chrome/browser/ui/helium/workspaces/` directory is absent. Existing
  `chrome/browser/ui/helium/` contains only the layout controller pair.
- No conflicting unrefreshed changes were found in the patch footprint,
  including the existing UI/GN seams. This is not a pristine-tree assertion:
  S has no Git baseline, and files outside the quilt footprint, generated
  resources, downloaded toolchains, and out/ were not exhaustively compared
  against the upstream archive. Before touching a previously unpatched
  existing file, compare it against its appropriate original source/resource
  baseline; stop if unexplained changes are found.

SHA256 provenance:

```text
df11b1f203e01804d0c25c43c21611a955ba9005b76a18435e8d4d02c8185b68  C/patches/series
8c93fc6aae5f05fb56c38380a46bf776676e598217d39801ae095d73d34bf531  P/patches/series.merged
8c93fc6aae5f05fb56c38380a46bf776676e598217d39801ae095d73d34bf531  S/.pc/applied-patches
d64dd81f92b5ea4593eafdcbb9d96aef9857c90eefa74339990e90895cc30db4  S/chrome/browser/ui/BUILD.gn
b742a76a09672721796bd07b4712006606c8a91436717e4466da886c09840471  S/chrome/test/BUILD.gn
```

Ephemeral evidence: `/tmp/i3-t01-replay-op_vm29u/audit.json` and
`/tmp/i3-t01-canonical-t_wb_5bk/audit.json`. Do not depend on these surviving;
the reproducible audit is below.

## Tooling boundary

`P/scripts/shared.sh` fixes `_main_repo` to T; `dev.sh` sources
T/devutils/set_quilt_vars.sh, then overrides QUILT_PATCHES and QUILT_SERIES to
P/patches and its series.merged. Merely sourcing dev.sh calls setup_environment
(which can mkdir a cache), so preflight did not source it.

Host `quilt` is unavailable. Existing local Docker image
`chromium-builder:trixie-slim` (`a8356d8b1656`) contains **quilt 0.68**.
Read-only container verification returned top
`helium/linux/disable-tab-strokes.patch` and “File series fully applied”.
S/.pc records container paths `/repo/patches` and
`/repo/patches/series.merged`; keep that mapping rather than rewriting metadata.
No containers were running at inspection time. The wrapper's source/environment
setup and `new/add/refresh/diff -z/pop -R/push` cycle were also exercised on a
scratch two-file tree: `SMOKE_PASS`, new-file recreation and edited contents
verified (`/tmp/i3-t01-quilt-smoke-c9OTTT`).

An attempted live `quilt diff -z` with a read-only source/root mount failed
because quilt needs scratch files under /tmp and the source directory. It made
no source changes and is not counted as a successful cleanliness check; the
independent 1,626-file replay establishes that evidence instead. Use the
read-only commands below only for metadata; use diff with the gated writable
wrapper or a scratch copy.

Read-only checks (safe to repeat):

```bash
git -C "$C" status --short
git -C "$T" status --short
git -C "$P" status --short
cmp "$C/patches/series" "$T/patches/series"
cmp "$S/.pc/applied-patches" "$P/patches/series.merged"
docker run --rm --network none --read-only --user "$(id -u):$(id -g)" \
  -v "$P:/repo:ro" -w /repo/build/src --entrypoint bash \
  chromium-builder:trixie-slim -c '
    export QUILT_PATCHES=/repo/patches
    export QUILT_SERIES=/repo/patches/series.merged
    quilt --quiltrc - top
    quilt --quiltrc - unapplied
  '
```

Do not run `he setup/reset/pull`, `he push` (uses `--refresh` globally),
`he pop` (whole stack), or `he unmerge` on this live merged tree.
`update_platform_patches.py unmerge` moves **every generic patch** back into
its script's checkout and rewrites series. It is not a narrow export command.
The safe adaptation below invokes its inspected Python function on disposable
copies only. This is deliberate, not a bypass via hand-edited patch hunks.

## Owned patches and placement

Reserve these generic names in execution order; create only when needed:

```text
helium/ui/workspaces/model.patch       # T02–T04 pure model and GN/tests
helium/ui/workspaces/geometry.patch    # T05
helium/ui/workspaces/views.patch       # T06–T08
helium/ui/workspaces/controller.patch  # T09–T12
helium/ui/workspaces/commands.patch    # T13
helium/ui/workspaces/tests.patch       # later integrated coverage
```

Append this block after `helium/ui/exclusive-access-bubble.patch` in C and T's
generic series. In the merged series it MUST remain before the first platform
patch. Putting a new patch on the current platform top would cause unmerge to
classify it as Linux-owned. Preserve all existing generic and platform order.
Do not refresh existing multi-contents/split-view/corners/separators/side-panel/
zen patches to absorb workspace changes.

## Future edit / refresh / verification sequence (not executed live)

Run one worker at a time; stop other source writers. Repeat status and source
checks first. Save outside all repositories a dated backup of current series,
owned patches, and every file you will edit, plus `.pc` and P/patches (an archive
is a recovery snapshot, not a directory synchronization). Record absent paths
so new files can be distinguished from pre-existing files. Do not proceed on
unexpected differences. Example snapshot:

```bash
B=$(mktemp -d /tmp/i3-workspaces-backup-XXXXXX)
tar -C "$P" -czf "$B/quilt-and-patches.tar.gz" patches build/src/.pc
# Add an explicit reviewed list of source paths for this task to the snapshot.
# For T02 the workspace files do not exist yet; save the GN files before editing.
cp "$S/chrome/browser/ui/BUILD.gn" "$B/ui-BUILD.gn"
cp "$S/chrome/test/BUILD.gn" "$B/test-BUILD.gn"
cp "$C/patches/series" "$B/canonical-series"
cp "$T/patches/series" "$B/tooling-series"
```

Use this host-shell wrapper for source mutations (only after the gate):

```bash
q() {
  docker run --rm --network none --read-only --tmpfs /tmp \
    --user "$(id -u):$(id -g)" -v "$P:/repo:rw" \
    -w /repo/build/src --entrypoint bash chromium-builder:trixie-slim \
    -c 'source /repo/helium-chromium/devutils/set_quilt_vars.sh
        export QUILT_PATCHES=/repo/patches
        export QUILT_SERIES=/repo/patches/series.merged
        exec quilt --quiltrc - "$@"' bash "$@"
}
# T02 only: remove nine platform patches, leaving the named patch applied.
q pop -R helium/ui/exclusive-access-bubble.patch
q new helium/ui/workspaces/model.patch

# Register BEFORE editing/creating. Adjust GN paths only after inspecting targets.
q add chrome/browser/ui/BUILD.gn chrome/test/BUILD.gn
mkdir -p "$S/chrome/browser/ui/helium/workspaces"
q add chrome/browser/ui/helium/workspaces/workspace_layout_model.h \
      chrome/browser/ui/helium/workspaces/workspace_layout_model.cc \
      chrome/browser/ui/helium/workspaces/workspace_layout_model_unittest.cc
# Edit these source files in S, never hand-author their exported diff.
q files
q diff --color=never
q refresh -p ab --no-timestamps --no-index --strip-trailing-whitespace
q diff -z --color=never  # must be empty: no unrefreshed delta
q push -a              # deliberately NO --refresh and NO force
```

For later edits to the same patch, `q pop -R helium/ui/workspaces/model.patch`
leaves it on top. For a new owned patch, pop to its intended generic predecessor,
then `q new` and `q add`. Never force a pop/refresh through overlapping changes.
Platform reapplication conflicts are blockers, not permission to refresh someone
else's patches. Inspect the patch and compare the full series, not just its name.

Before claiming an export reproduces tested source, at the owned-patch top save
SHA256 for every `q files` path, run `q pop -R` (one owned patch), `q push` (one),
and compare those hashes. Reapply the platform suffix with `q push -a`, repeat
the patch-footprint audit, and run task-appropriate builds/tests on that final
state. A quilt success does not establish C++ correctness or binary freshness.

## Narrow export via verified scratch unmerge

The script classifies patches encountered before the first original platform
entry as generic, then restores saved series comments. A scratch routing probe
with `helium/ui/workspaces/model.patch` before the platform block succeeded;
it appeared only in the generic output at
`/tmp/i3-t01-export-d43qdnj8`. No real checkout was unmerged.

After refresh and source verification, project the live merged stack into a
fresh disposable directory (host Python, no source writes):

```bash
E=$(mktemp -d /tmp/i3-workspaces-export-XXXXXX)
export E T P
python3 - <<'PY'
import os, shutil, sys
from pathlib import Path
E, T, P = (Path(os.environ[k]) for k in ('E', 'T', 'P'))
sys.path.insert(0, str(T / 'devutils'))
from update_platform_patches import unmerge_platform_patches
shutil.copytree(P / 'patches', E / 'platform')
(E / 'generic').mkdir()
assert unmerge_platform_patches(E / 'platform', E / 'generic')
PY
# Read E/generic/series and the ENTIRE owned patch before copying.
# For T02, require its series to equal C's prior series plus exactly one line.
# Compare platform series to P/patches/series.orig: ordering must not change.
p=helium/ui/workspaces/model.patch
cmp "$E/generic/$p" "$P/patches/$p"
# For a new patch, require destination absence. For a refresh, back it up and
# require C and T's prior copies to match before either is overwritten.
mkdir -p "$T/patches/helium/ui/workspaces" "$C/patches/helium/ui/workspaces"
cp "$E/generic/$p" "$T/patches/$p"
cp "$T/patches/$p" "$C/patches/$p"
cmp "$E/generic/$p" "$T/patches/$p"
cmp "$T/patches/$p" "$C/patches/$p"
sha256sum "$P/patches/$p" "$T/patches/$p" "$C/patches/$p"
```

For each NEW patch only, append its single approved entry to C and T's series
(after confirming both previous series match and the entry is absent):

```bash
cmp "$C/patches/series" "$T/patches/series"
! grep -Fxq "$p" "$C/patches/series" || { echo 'already registered'; exit 1; }
printf '%s\n' "$p" >> "$T/patches/series"
printf '%s\n' "$p" >> "$C/patches/series"
cmp "$C/patches/series" "$T/patches/series"
cmp "$C/patches/series" "$E/generic/series"
git -C "$T" diff --stat
git -C "$C" diff -- patches/series
git -C "$C" diff --check
```

If series comparison fails (including unexpected metadata changes), stop and
inspect; do not copy a whole series over it blindly. For existing patches the
series must remain unchanged. Confirm only the approved owned names changed
in T and C; all existing non-owned patch hashes must still match the snapshot.
New untracked patches require explicit inspection since plain `git diff` omits
them. Stage only named task files and `patches/series`, inspect the cached diff,
then commit in C. T is a transport working tree; record its intentional owned
changes for the next worker rather than resetting or pulling it.

The live P series remains merged. Its original `series.prepend` need not be
rewritten: the inspected unmerge logic recognizes new generic entries before
the platform boundary. Never use the scratch operation to migrate unrelated
refresh churn back into either real shared checkout.

## Reproducible source-footprint audit

This writes only a new scratch directory. Empty earliest backups represent
new or empty pre-patch files; both successful full replays were compared with
live file existence and contents. It does not scan files quilt never owned.

```bash
export C P S
python3 - <<'PY'
import os, shutil, subprocess, tempfile
from pathlib import Path
C, P, S = (Path(os.environ[k]) for k in ('C', 'P', 'S'))
patches = (S / '.pc/applied-patches').read_text().splitlines()
for mode in ('merged', 'canonical'):
    W = Path(tempfile.mkdtemp(prefix='i3-audit-' + mode + '-'))
    seen = set()
    for name in patches:
        root = S / '.pc' / name
        for f in root.rglob('*'):
            if not f.is_file() or f.name == '.timestamp':
                continue
            rel = f.relative_to(root)
            if rel in seen:
                continue
            seen.add(rel)
            (W / rel).parent.mkdir(parents=True, exist_ok=True)
            if f.stat().st_size:
                shutil.copyfile(f, W / rel)
    for name in patches:
        patch = P / 'patches' / name
        if mode == 'canonical':
            patch = C / 'patches' / name
            if not patch.exists():
                patch = W / 'platform-input.patch'
                patch.write_bytes(subprocess.check_output([
                    'git', '-C', str(P), 'show', 'HEAD:patches/' + name]))
        subprocess.run(['patch', '--batch', '--forward', '--fuzz=0', '-p1',
                        '-d', str(W), '-i', str(patch)], check=True)
    differences = [str(f) for f in sorted(seen)
                   if (W / f).exists() != (S / f).exists()
                   or ((W / f).exists()
                       and (W / f).read_bytes() != (S / f).read_bytes())]
    print(mode, W, len(patches), len(seen), differences)
    assert not differences
PY
```

The canonical pass is meaningful only once every owned patch is exported; do
not misclassify an as-yet-unexported generic patch as a platform patch. Changes
to platform HEAD or legitimate source generation require an explicitly updated
baseline, not silently weakened comparisons.

## Recovery and handoff

- On unexpected delta/conflict, stop, retain logs and snapshots, and report the
  exact file/patch. Do not run forced quilt operations, git clean/reset, setup,
  pull, whole-directory rsync, or substitutions to make checks pass.
- Before refresh, save `q diff --color=never` and copies of changed source files
  into B. Resolve only reviewed task-owned edits; never discard unknown edits.
- A failed ordinary push/pop may leave a partially advanced stack. Inspect
  `q top`, `q applied`, `q unapplied`, `.pc/applied-patches`, and rejects before
  deciding the next operation. Do not assume the old top is still current.
- To recover an exported patch after a mistaken copy, compare with B and restore
  only that named destination and the reviewed series line. To undo source work,
  first preserve it, then remove edits deliberately or refresh the owned patch
  as a recoverable backup; only a clean `q pop -R` may remove it. Re-push the
  saved owned patch to return to the recorded state. Restoring `.pc` alone is
  unsafe: it must correspond exactly to source and patch files. Never unpack
  the broad snapshot over live state without a separate reviewed recovery plan.
- No commit skill was found in the available skills or the local skill search;
  follow CONTRIBUTING.md's scope-first commit style and explicit path staging.

**T02 readiness:** approved source footprint is unambiguous; no conflicting
source changes found. Use the existing container quilt, insert before Linux
patches, add files before editing, export only the new model patch, and verify
source/patch identity again before tests. T02 was not executed. No claim is
made about compilation, existing binary freshness, browser behavior, or ISC
feature completion. This preflight supports patch safety and ISC-A-4 only.
