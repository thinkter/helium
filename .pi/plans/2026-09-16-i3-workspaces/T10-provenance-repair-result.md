# I3-T10 provenance repair result

Date: 2026-09-17
Scope: repair of patch/quilt/source provenance only. No feature implementation,
no source-content edits, no build, test, browser run, resource restoration,
download, setup, reset, pull, or broad synchronization.

## Authority and starting state

| Location | Role | Revision / state at repair |
| --- | --- | --- |
| `C=/home/ashman/Documents/rubbish/helium` | canonical | `14d7630` |
| `P=/home/ashman/Documents/rubbish/helium-linux` | live merged stack | `2686b7c` |
| `T=P/helium-chromium` | transport working tree | dirty by design (series + workspace patches) |
| `S=P/build/src` | applied Chromium source | no git; 1643-footprint tracked by quilt |

The corrected merged series is exactly **349 entries**: 340 generic
(`C/patches/series`, ending at `helium/ui/workspaces/controller.patch`) plus
9 platform entries (`P/patches/series.orig`), in the existing order
(`drop-nodejs-version-check`, `use-oauth2-client-switches-as-default`,
`fix-compiling-on-arm64`, `change-chromium-branding`, `rename-chrome-binary`,
`use-default-theme`, `add-middle-click-paste-flag`,
`add-error-for-missing-desktop-file`, `disable-tab-strokes`). The count was
verified against both series files and against the 349 applied quilt entries;
`controller.patch` is entry 340 (1-based) and the first platform patch is
entry 341.

## Defects confirmed

1. **`controller.patch` was refreshed against a mixed, corrupt quilt
   baseline.** All three copies (C, P, T) held the same 2137-line file,
   sha256 `94cda3530cfdad65410b33f9ad599ec08239c0e67a959f024adf4bba16e735dc`,
   17 files / 73 hunks / +1476/-46. Its `.pc` backups were wrong in both
   directions:
   - `chrome/browser/ui/tabs/tab_strip_model.cc` and `.h` were byte-identical
     to upstream, missing earlier Helium changes (227 and 23 added lines
     respectively), so the refresh re-added changes owned by
     `add-flag-to-close-window-with-last-tab`, `hibernate-tab-context-menu`,
     `tab-cycling-mru-impl`, `close-tabs-to-left` and others.
   - `chrome/browser/ui/views/frame/browser_view.cc` and `.h` already
     contained controller-owned hunks (19 and 12 lines), so the refresh
     emitted those changes in the wrong place.
   Replay stopped at entry 340 with reversed/failed hunks. The remaining 13
   controller backups were already correct.

2. **`disable-tab-strokes.patch` was contaminated.** P held a 140-line file,
   sha256 `bad65d55bacbb1d34a12de37f9d2e08ee0de135712fead5898194dd89f6000d0`,
   containing controller hunks in `browser_view.cc`. The pre-accident bytes
   were preserved by quilt in `patches/helium/linux/disable-tab-strokes.patch~`
   and are byte-identical to `P` HEAD: 14 lines, sha256
   `84da2e6360ccfb242fc54c2c4832416b6bd6f10e0915c7970ad09e5a884f310d`. The
   contaminated file never applied cleanly in reverse; its content was a
   refresh artifact, not live source state.

3. **`S/.pc/applied-patches` had 348 entries** (sha256
   `00f03cb2e3a2c96064ed3d1a2014629214e0e3c5c9f3e4722f296820e00d1db3`),
   missing the top `disable-tab-strokes` entry even though its change was
   live, and `S/.pc/helium/linux/disable-tab-strokes.patch/` did not exist.

4. Unrelated recovery artifact `S/chrome/browser/ui/views/frame/browser_view.cc.orig`
   (timestamp 00:24) remains in place; it was not created or touched by this
   repair.

## Method (no hand-authored hunks, no blind `.pc` overwrite)

Snapshots were taken before any mutation and are retained under
`/home/ashman/Documents/rubbish/tmp-audit/repair-snapshot/`:

- `patches-full-hashes.txt` — all 697 files under `P/patches`.
- `pc-full-hashes.txt` — all 2802 files under `S/.pc`.
- `source-footprint-hashes.txt` — the 1643 live source files owned by the
  patch footprint.
- `pc-controller.tar.gz`, `canonical-controller.tar.gz`,
  `patches-tracked-platform.tar.gz` and `corrupt-copies/` — byte-level
  recovery copies of the corrupt patches and controller `.pc`.
- `snapshot-meta.txt` — C/P HEADs and timestamp.

Reconstruction (`tmp-audit/provenance-repair/`):

1. A pristine upstream baseline was rebuilt from the earliest quilt backup of
   every file across the 349-entry stack (1643 files; zero-size backups are
   files created by their first touching patch).
2. The pre-controller state was produced by applying the 339 verified
   pre-controller patches with `patch --batch --forward --fuzz=0`.
3. The desired controller-end state was produced from the live source by
   reverse-applying the platform suffix in reverse order, using the restored
   14-line `disable-tab-strokes.patch` and the 8 other platform patches.
4. The diff between those states is **exactly the 17 files** below (count
   verified independently of `T10-provenance-diagnosis.md`, whose file list is
   therefore confirmed; only its hunk order differs):
   `chrome/browser/ui/BUILD.gn`,
   `chrome/browser/ui/browser_window/internal/browser_window_features.cc`,
   `chrome/browser/ui/browser_window/public/browser_window_features.h`,
   `chrome/browser/ui/helium/workspaces/workspace_controller.cc` (new),
   `chrome/browser/ui/helium/workspaces/workspace_controller.h` (new),
   `chrome/browser/ui/helium/workspaces/workspace_controller_browsertest.cc` (new),
   `chrome/browser/ui/tabs/tab_strip_model.cc`,
   `chrome/browser/ui/tabs/tab_strip_model.h`,
   `chrome/browser/ui/views/frame/browser_view.cc`,
   `chrome/browser/ui/views/frame/browser_view.h`,
   `chrome/browser/ui/views/frame/contents_border_controller.cc` (upstream file, modified),
   `chrome/browser/ui/views/frame/workspace_contents_view.cc`,
   `chrome/browser/ui/views/frame/workspace_contents_view.h`,
   `chrome/browser/ui/views/frame/workspace_contents_view_browsertest.cc`,
   `chrome/browser/ui/views/frame/workspace_resize_area.h`,
   `chrome/browser/ui/views/frame/workspace_tabbed_header_view.cc`,
   `chrome/test/BUILD.gn`.
5. A **fresh scratch quilt stack** was created: the 339 verified pre-controller
   `.pc` backups plus the pristine footprint were seeded, then
   `quilt new helium/ui/workspaces/controller.patch`, `quilt add` of the 17
   files, working-tree copy of the desired content, and
   `quilt refresh -p ab --no-timestamps --no-index --strip-trailing-whitespace`.
   `quilt diff -z` was empty. The scratch series was then extended to all 349
   entries using the restored 14-line `disable-tab-strokes.patch` and the
   8 platform patches; `quilt push -a` applied all 9 with `quilt diff -z`
   empty.
6. The scratch result of the full 349-patch quilt stack equals the live source
   across all 1643 footprint files byte-for-byte (0 differing, 0 extra).
7. The diagnosis candidate `/home/ashman/Documents/rubbish/tmp-audit/correct-controller.patch`
   was independently checked: it applies to the same baseline and produces
   byte-identical output to the quilt-generated patch (48 hunks,
   +1255/-36); only the file/hunk order differs. It was not copied blindly.

## Repairs applied (exact)

| Artifact | Before | After |
| --- | --- | --- |
| `C/P/T patches/helium/ui/workspaces/controller.patch` | `94cda353…e735dc` (2137 lines, 73 hunks, +1476/-46) | `e3e011f2602a86e6831c570d7e53441d34147b1009e506d72d8e179636c14643` (1700 lines, 48 hunks, +1255/-36) |
| `P patches/helium/linux/disable-tab-strokes.patch` | `bad65d55…f6000d0` (140 lines) | `84da2e6360ccfb242fc54c2c4832416b6bd6f10e0915c7970ad09e5a884f310d` (14 lines, byte-identical to `~` backup and P HEAD) |
| `S/.pc/…/controller.patch/chrome/browser/ui/tabs/tab_strip_model.cc` | `13e46661…ab832c` (upstream) | `4c9551581c7f56309e500b30c4b1817fb3d489738aa922b7b621d151e3d0e793` |
| `S/.pc/…/controller.patch/chrome/browser/ui/tabs/tab_strip_model.h` | `d2812289…e851026` | `84233e8be1a9ac003977be232ed604b8044a648b1b78f166004ca426f854a10f` |
| `S/.pc/…/controller.patch/chrome/browser/ui/views/frame/browser_view.cc` | `4e656886…35ef98` (contained controller hunks) | `3334a1d8d560759ad420f8fccee62357ac27d47c26762501254b8ae7766d676d` |
| `S/.pc/…/controller.patch/chrome/browser/ui/views/frame/browser_view.h` | `49dbc52f…c35ca0` (contained controller hunks) | `7262030b045983b619b6e3d39b5e0e24e22d47326006654ba94aba8d20d64588` |
| `S/.pc/helium/linux/disable-tab-strokes.patch/` | absent | created with `chrome/browser/ui/views/frame/browser_view.cc` = `1a1eab4105c06057173e855ce4d082f4f092a7dddf13fdc6bea97c857a30ec06` (pre-stroke state) and `.timestamp` |
| `S/.pc/applied-patches` | `00f03cb2…e00d1db3` (348 entries) | `c19da5c72252037e80e8fd77d115967d174d023c869548bcee8f65b7a4e1c24b` (349 entries, byte-equal to `P/patches/series.merged`) |

The other 13 controller `.pc` backups were byte-identical to the verified
pre-controller state and were left unchanged. All other `.pc` platform
backups (8 patches) also matched the verified scratch state and were left
unchanged. Only the four listed backups plus `applied-patches` hash-changed,
and only the `disable-tab-strokes` `.pc` directory was added.

Export followed the workflow: a scratch unmerge projected `P/patches` into
`E/generic` + `E/platform`. `E/generic/series` equals the canonical
`C/patches/series` entry list (340), `E/platform/series` equals
`P/patches/series.orig` (9), `controller.patch` routed generic-only and
`disable-tab-strokes.patch` platform-only, and the corrected patch was copied
to `T` and `C`. `C`, `T` and `P` controller hashes are now identical.

## Verification evidence

- **Full merged replay (plain GNU patch, the workflow's audit):** the pristine
  baseline was rebuilt from the repaired live `.pc` and is identical to the
  pre-repair baseline (0 differing files), then all 349 live `P/patches`
  entries were applied with `--batch --forward --fuzz=0`: **0 failures**,
  all 1643 live footprint files byte-equal, 0 extra files.
- **Full canonical replay:** canonical `C/patches` generic entries (340,
  including the corrected controller) plus the original `P` HEAD platform
  patches (9): **0 failures**, all 1643 footprint files byte-equal, 0 extra.
- **Scratch quilt stack:** 349 applied, `quilt diff -z` empty, scratch source
  equals live source for all 1643 files.
- **Live quilt state:** `top` = `helium/linux/disable-tab-strokes.patch`,
  349 applied, “File series fully applied”, `quilt diff -z` empty. An
  integrity manifest of 5255 `.pc` + source entries was unchanged by that
  command.
- **Live `.pc` correspondence:** all 349 patch directories and all 2451
  backup files match the verified scratch metadata exactly.
- **Source preservation:** all 1643 footprint source hashes are unchanged
  from the pre-mutation snapshot (0 differences).
- **Non-owned patch preservation:** the `P/patches` hash manifest shows
  exactly two changed files (`controller.patch`,
  `disable-tab-strokes.patch`); the other 695 files are unchanged, with no
  additions or removals.

## Limitations and handoff

- The upstream baseline is derived from the earliest quilt backups rather than
  a pristine Chromium checkout; correctness is established by the two
  independent complete replays reproducing the live tree exactly.
- The stray `browser_view.cc.orig` artifact was left in place deliberately.
- `T` remains a dirty transport tree (its series modification and workspace
  patches are intentional and were not committed here); `P` generic patch
  copies remain untracked by design.
- `.pc/.timestamp` mtimes were refreshed for the two repaired patches so quilt
  does not report them as modified.
- No build, runtime, browser, resource or test verification was performed or
  claimed. T10 feature status is unchanged and remains incomplete; the plan’s
  feature checklist was not touched.
- Ephemeral evidence (scripts, logs, scratch trees) lives under
  `/home/ashman/Documents/rubbish/tmp-audit/provenance-repair/` and
  `tmp-audit/repair-snapshot/`.
