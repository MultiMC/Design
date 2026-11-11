# Updater v2 review: can an install be stranded?

A working map for going through the update flow together, step by step: what is read, how
it is parsed, what is decided, and what a failure leaves behind. The question at every step:
can a Launcher end up somewhere it no longer updates from, by our mistake or by bad luck?

Prepared 2026-11-10 from the `updater` branch (with the staged variant leniency) and hopper
as of today. Nothing here is confirmed until we've gone through it together.

**This review blocks the cutover.** Every step must be confirmed and every gap (G1–G12)
fixed or deliberately accepted before production Launchers are moved to the new updater.

Marks:

- **ok**: read in the code and looks safe; confirm together.
- **gap**: a way to strand installs (or to hide that they are stranded) that I found; decide
  together whether it's real and what to do.
- **check**: needs a test or a look at something outside the code.

Paths are from the MultiMC directory.

## 0. What "stranded" means

An install is stranded when it never gets to a newer build again without the user
reinstalling by hand. Worse is stranded *silently*: nothing tells the user. A check that
fails today and works tomorrow is not stranded; one that fails every day is.

## 1. What the Launcher knows before it asks anything

| What | Where it comes from | If it's wrong or missing |
|---|---|---|
| Updater enabled | `Launcher/buildconfig/BuildConfig.cpp.in:47`: refspec starts with `refs/heads/`, updater configured (root keys, keys floor, variant), platform set, build number ≥ 0 | Updater off for good: **gap G7** |
| Build number | `Launcher_VERSION_BUILD`, from hopper's build counter | Wrong number: tier decisions are wrong |
| Variant ID | `Launcher_UPDATER_VARIANT`, from hopper's platform table (`hopper/lib/common.py:68`), also used for the manifest | Mismatch: "no version for this installation". One source for both, so **ok** by construction, but nothing checks the binary (G7) |
| Pinned root keys and keys floor | CMake, from the setup; `State::State` (`Launcher/launcher/updater/v2/State.cpp:25`) refuses a floor that doesn't list exactly the pinned keys | Config error: every check fails with it. The gate checks the keys are in the binary (`hopper/launcher/compat-check.py`, `check_updater_keys`) |
| Serial floor | CMake, always 0 (hopper doesn't set it) | Only weakens replay protection for fresh installs, **ok** |
| Stored state: highest serial, newest keys document, newest mirror list | `updater/state.json` in the data directory; `State::load` (`State.cpp:44`); each part on its own, broken parts fall back to the floors | **ok**. Deleting it is a way out of a serial problem (G2) |
| Tier | `UpdateTier` setting; empty starts the setup wizard (`Application.cpp:1060`); unparsable means Stable (`MainWindow.cpp:1395`) | **ok** |
| Preferred variant | `UpdateVariant` setting | Ignored when the build doesn't have it or it doesn't run here (`Plan.cpp:72`), **ok** |
| AutoUpdate | Setting, default on (`Application.cpp:666`) | Off means no checks unless asked: the user's choice |
| The system | `SystemRequirements::detect()` (`SystemRequirements.cpp:264`) | Wrong values make requirements look unmet: **check G8** |

## 2. When a check runs

- Once at startup (`MainWindow::autoCheckForUpdates`, `MainWindow.cpp:2240`), if AutoUpdate
  is on and no instance is running. Also on "Check for updates" and on a tier or variant
  change in the settings (`UpdatesPage.cpp:129`, with `tierSwitched`).
- No periodic check: a Launcher left running for weeks checks once. That's a delay, not a
  strand.
- A result that arrives while an instance runs is dropped (`MainWindow.cpp:1411` and the
  same check per case). With `-l` launches at startup, the update shows on the next start.
  Delay only.

## 3. The check (`Launcher/launcher/updater/v2/Checker.cpp`)

Every document comes with its `.sig` from the same mirror. The signature is checked over the
raw bytes before parsing. A mirror that fails or serves something unacceptable is skipped
for the next one (`fetchFrom`, `:164`).

**3.1 Mirrors to try.** `Net::Mirrors::candidates` (`Launcher/launcher/net/Mirrors.cpp:114`):
the signed mirror list's groups first, then the compiled-in base URLs that the list doesn't
already include. A bad or empty mirror list can't remove the compiled-in hosts. Unreachable
hosts are tried last for 5 minutes. **ok**

**3.2 Keys chain** (`nextKeys`, `:225`). Starts from the stored or floor keys document,
version N. Fetches `keys/N+1.json`, checks it against N's root keys (one valid line by a
trusted key is enough, and every trusted line must verify: `Formats.cpp:555`), then repeats.
"Not found" on every mirror ends the walk. Anything else that fails on every mirror fails
the check.
- A malformed published keys document fails every check until it's fixed on the server.
  Fixable, but silent (G1).
- **gap G3:** if the newest keys document has expired (`afterKeys`, `:254`), every check
  fails. That covers keys not renewed in time, and a user clock that is far ahead.
- Rule to confirm: a keys version number is never reused with different content. A Launcher
  that stored the first one never reads the second.

**3.3 Mirror list** (`:273`), signed by an online key. An older list is ignored, not refused.
**ok**

**3.4 Tags** (`:283`), signed by an online key.
- Header: `type` and `format` must be exact, which is the only strict part (`header`,
  `Formats.cpp:93`). Unknown fields are ignored (`Json::requireKeys` checks only for
  presence). Rule to confirm: format 1 forever on this tree. Another format means another
  tree, or every client is stranded (**G12**).
- A serial lower than the stored one is refused on that mirror (`:289`). A mirror that lags
  behind is skipped, **ok**.
- **gap G2:** if *every* mirror serves a lower serial, the check fails, and keeps failing
  until the published serial passes the stored one. See 6.3.

**3.5 Tier decision** (`decideTier`, `Plan.cpp:37`). Newer means Update. Same means
NoUpdate. Older means BehindInstalled, or Downgrade if the user just switched tiers. A
withdrawn installed build only gets a message (`:336`); it stays until the next build.
**ok**

**3.6 The tier's build manifest** (`:349`).
- Build-level errors fail the check: header, build number, commit, no variants, the
  supersedes rules (`parseBuild`, `Formats.cpp:481`). Moving the tag fixes it, since
  manifests aren't changed in place.
- A variant this Launcher can't read is left out (`Formats.cpp:505`). **ok**, done today.

**3.7 Variant choice** (`chooseVariant`, `Plan.cpp:68`).
1. The preferred variant, if present and met.
2. Otherwise, the furthest successor along `supersedes` whose requirements are met.
3. Otherwise, its own variant if met.
4. Otherwise, NotForThisMachine, with the reasons.

Unknown requirement keys count as unmet (`SystemRequirements.cpp:350`).
- NotForThisMachine is shown once per build, even without asking (`MainWindow.cpp:1521`).
  **ok**: not silent.
- NoUpdate with no variant found also ends in NoUpdate (`:378`). **ok**

**3.8 The installed build's manifest** (`:402`). Optional: if it's purged or withdrawn, the
update still runs. It only can't remove files the next build doesn't have (stale files
stay). **ok** (stale Qt plugins: **check**, minor)

**3.9 Result handling** (`MainWindow::updateCheckedV2`, `MainWindow.cpp:1402`).
- **gap G1:** a Failed result from an automatic check is never shown (`:1526`, `notify` is
  false). Every gap that fails checks repeatedly (G2, G3, a malformed document, blocked
  hosts) is therefore silent. Only someone who clicks "Check for updates" sees the reason.

## 4. Download and install

**4.1 Plan** (`planInstall`, `Launcher/launcher/updater/v2/Install.cpp:87`).
- Removals first: files and links of the installed variant that the next one doesn't have.
- Then placements: files whose content differs. Unchanged files only get a fix to their
  executable bit.
- Then links.
- **gap G4 (check):** a case-only rename on a case-insensitive file system (Windows, macOS).
  Build N has `Foo.dll` and N+1 has `foo.dll` with the same content. The plan finds `foo.dll`
  already correct (it opens `Foo.dll`), so it places nothing. It then removes `Foo.dll`, the
  same file, and the result is a missing DLL. The manifest checks only look at case within
  one manifest, not between builds. Needs a test.

**4.2 Download** (`DownloadTask.cpp`).
- Content files are addressed by SHA-256. They go through the mirrors, with the size and
  hash checked.
- Everything goes into `update/XXXXXX` before anything changes, so a failure here changes
  nothing. **ok**

**4.3 Apply** (`InstallController::installUpdates`, `InstallController.cpp:32`).
- First the previous backup (`update/backup`) is deleted.
  - **gap G9:** if that fails, every update fails until it's cleared by hand. The user is
    told (in a modal), so it's not silent.
- Each change moves what was there into the backup, then moves the new file in. If a change
  fails, everything is rolled back in reverse order.
- Windows XP: the exe is swapped by a VBScript after exit, with no error handling. That path
  is XP only.
- **gap G5:** an apply that's interrupted (a crash, power loss) leaves mixed files, and
  there's no journal and no recovery on the next start. If the Launcher still starts, the
  next update repairs it, because the plan compares hashes. If it doesn't, it needs a manual
  reinstall. The backup is there, but nothing uses it.

**4.4 Live check** (`InstallController.cpp:127`, `LiveCheck.cpp`, `Application.cpp:643`).
- The new process writes `live.check` early, before settings load. The old one waits 30 s
  and offers a rollback if it doesn't see it.
- **gap G6:** this proves the new build *starts*, not that its *updater works*. A build
  whose updater is broken passes, and its installs are stranded. This is the bad-bridge
  question in general form, and it applies to every build, not only the bridge.

## 5. The old layout and the bridge

- Old Launchers run *their own* old code: v1 `UpdateChecker`, always the `develop` channel,
  any build number different from their own (`UpdateChecker.cpp:162`), their platform
  through their own `getIdealPlatform`. The oldest ones read the `stable` channel frozen in
  2023 and take two hops.
- The bridge goes into each old-layout platform directory (lin64, lin32, win32, win32-5.15.2,
  osx64-5.15.2). Each copy must be the variant matching that directory, so the bridge looks
  for the right variant in the new tree.
- The bridge is applied by the *old* v1 apply code. That's the same path as every release
  since 2023, so the rehearsal covers it.
- **G10 (known):** hopper doesn't enforce the freeze yet (Updater.md, "The cutover"). This
  has to happen before the cutover.
- The open question from Updater.md: how to rule out a bad bridge before the freeze (G6,
  stricter).

## 6. Server side (hopper)

**6.1 Publishing order** (`hopper/files/publish.py:52`). Content files, then manifests, then
keys documents and the mirror list, then the tags document last. Each file is replaced
atomically. **ok**

**6.2 Deploy** (`hopper/updater/deploy.py`). The whole tree is validated before writing and
again after. **ok**
- The gate checks requirements and keys, but nothing about the updater's configuration in
  the binary. **gap G7** (below).

**6.3 Serial** (`hopper/lib/updater_tree.py:157`): one higher than *the tree's* current tags
document.
- **gap G2:** restoring a snapshot (`files/snapshots.py restore`) and publishing lowers the
  published serial below what Launchers have stored. They refuse every tags document until
  the serial is higher again: one step per deploy or tag move, silently (G1). Nothing in
  hopper remembers the highest serial ever published.

**6.4 Tag, remove, prune.** Tags only move forward, except remove, which writes a
higher serial. Stable can't be removed. Prune never touches tagged builds or past Stable
builds, and a purged build still updates. **ok**

**6.5 Keys.**
- Online key lost: a new keys document signed by a root key fixes it. **ok**
- Both root keys lost: once the newest keys document expires, every install is stranded for
  good. That's why there are two keys and the ceremony.
- Expiry: G3, with the 18-month reminder.

**6.6 Caching** (**check G11**). Cloudflare could serve a stale `tags.json` with a fresh
`.sig`, or the reverse. That fails verification on that host, and if every mirror sits behind
the same cache, the check fails until it expires. It's temporary, but worth knowing what the
cache rules are for the tree's documents.

## 7. The list

| # | What | Kind | Silent? |
|---|---|---|---|
| G1 | Failed automatic checks are never shown | makes the others silent | — |
| G2 | Serial goes back after a snapshot restore; Launchers refuse tags until it passes | server mistake | yes (G1) |
| G3 | Newest keys document expired (or user clock far ahead): every check fails | server or user clock | yes (G1) |
| G4 | Case-only rename between builds removes a file on Windows and macOS | install | broken install |
| G5 | Interrupted apply: mixed files, no recovery on start | bad luck | if it doesn't start |
| G6 | Live check proves a start, not a working updater (bad bridge, any build) | build | yes |
| G7 | Nothing checks that a released binary has its updater enabled, its variant, its number | build | yes |
| G8 | macOS version as seen by an old-SDK build (10.16 on 11+) can make future requirements unmet | requirements | no (NotForThisMachine) |
| G9 | Previous backup can't be deleted: every update fails | install | no |
| G10 | Freeze of the old layout not enforced yet | cutover | — |
| G11 | Cache serving a mismatched document and signature | serving | temporary |
| G12 | Format number is strict: never change it on this tree | rule | — |

Ideas to talk about, not decided:

- **G1:** show an automatic check's failure when it repeats, for example after several days
  of failures, in the status bar or as a one-time notice.
- **G2:** hopper keeps the highest published serial outside the snapshots, and refuses to
  publish a tags document that isn't above it.
- **G6/G7 together:** a Stable promotion requires that the build itself updated successfully
  to a later build (on files2 or in Latest). The gate could also run the binary to print its
  updater configuration (enabled, variant, build, keys) and compare it with the manifest.
- **G4:** a test, then either handle it in `planInstall` or have the publisher refuse
  case-only renames between consecutive builds.
