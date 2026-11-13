# Updater v2 review, second pass

A second, independent pass over the new updater and its cutover plan, with the same
question as [UpdaterReview.md](UpdaterReview.md): can an install end up somewhere it no
longer updates from? It checks that review's gaps (G1–G12) against the code and adds what it
missed (N1–N9).

Prepared 2026-11-11 from the Launcher's `updater` branch (695b97e4), hopper's `master`
(80046cc) and this repo (ed30c8a). Read: `launcher/updater/v2`, `net/Mirrors.cpp`, the wiring
in `Application.cpp` and `MainWindow.cpp`; hopper's `updater/`, `lib/updater_tree.py`,
`lib/updater_formats.py` (the tree check), `files/publish.py`, `files/snapshots.py`, the
release job and the README's ceremony and cutover parts; Updater.md and UpdaterFormats.md.
Not read line by line: v1's code, `SystemRequirements.cpp`'s detection helpers, the
download archives, the web UI. Nothing was built or run except the two checks marked
"tried".

Paths are from the MultiMC directory.

## The short version

The design holds: signature first, content addressing, the write order and the "whole tree
checked before and after" gate are all as described, and I found no way for a hostile
mirror or a half-written publish to strand anyone. The first review's gaps are all real.

What can still strand installs is around the edges, in three groups:

1. **The cutover as the job is written today** makes the first production build with the new
   updater the bridge, for everyone, at once, before that build's updater has run against
   production (N1). This is the one way to strand everybody for good.
2. **Mistakes on the server that every Launcher refuses, silently.** A serial that goes back
   (N3; files2's tree was restarted this way, on purpose), a build an older Launcher can't
   apply (N4, G4) or doesn't find its variant in (N5), a host that stops answering 404 (N6).
   All can be fixed from the server, but only if someone notices, and failed automatic
   checks are never shown (G1).
3. **Nothing tests an update before a tag moves.** Every case in group 2, and a build whose
   own updater is broken (G6, G7), would be caught by one automated test: install the
   previous build, update to the new one, check again (N5).

## Must be settled before the bridge build

### N1. The first production build with updater 2 becomes the bridge by itself

- The release job (`hopper/laminar/cfg/scripts/release:62-66`) runs `files/deploy.py` and
  `files/publish.py` first, the current layout, and only then `updater/deploy.py` and a
  second publish. Updater.md ("The cutover", Order) says the opposite: the bridge is Stable
  in the new layout before it is published in the current one.
- Nothing in hopper knows about the freeze (G10): no setting, no refusal. `grep -i freez`
  finds nothing.
- So the moment production's settings say `UPDATER_VERSION=2`, the next push to `develop`
  is published to every v1 Launcher. They all move to it, and from then on they run its
  updater. If that updater fails in production (a wrong URL, key or variant, a platform
  where it can't apply), they are stranded and nothing reaches them. This is Updater.md's
  open question "If the bridge build turns out bad", and the job makes it the default
  outcome of flipping the setting.
- The files2 rehearsal can't rule it out: it runs with files2's keys, hosts and numbering.
  What is specific to production (the root keys and floor, the site URLs, the production
  tree, Cloudflare in front of it) is exactly what the bridge would run for the first time.

A way to close it, using only what exists:

1. **Freeze first, at the last v1 build.** hopper gets the freeze setting before anything
   else changes, and `files/deploy.py` refuses Launcher builds above it. From then on
   production builds go only into the new tree.
2. **Then turn updater 2 on.** Builds B, B+1, … are in the new tree only, with production's
   keys and hosts. No v1 Launcher sees them.
3. **Prove a candidate from both sides.** Install B from the tree's own download archives
   (`downloads/updater/`), on all five variants (Windows XP included), and update it to B+1
   through the real hosts. That shows B's updater works in production, which the live
   check can't (G6), and that B is applied by a v2 Launcher.
4. **Only then publish B into the current layout,** by an explicit command that names the
   build (the freeze moves to B). The v1 hop itself is the path every release since 2023
   has taken, and the files2 rehearsal covers it.

This needs the freeze to be a refusal with one exception, not a switch. It also settles
N8.

Agreed 2026-11-11, with the platforms tested side by side and the bridge going to all of
them at once (no delay between platforms). Done in hopper the same day (staged, not
committed): `CURRENT_LAYOUT_FROZEN_AT`, `files/deploy.py --bridge` and `--skip-frozen` for
the release job. The rules are tested; a bridge deploy that goes through isn't, until the
files2 rehearsal.

### G1, raised: tell the user when checks keep failing, and put it in the bridge

Confirmed: `MainWindow.cpp:1548` returns for every result of an automatic check except
NotForThisMachine, Failed included.

Every other gap here ends in "the check fails, silently". One small thing turns all of them
from stranded into "asked to reinstall": the Launcher remembers when a check last got a
verified answer, and after some days without one (14?) says so once, with the reason of the
last failure and a link to the download page. It needs nothing from the tree to work, which
is the point. It has to be in the bridge build, since a bridge without it can't be given it
later if its updater is the thing that's broken.

The same timestamp would let the settings page show "last checked", which makes a quiet
failure visible to anyone who looks.

Done in the Launcher 2026-11-11 (`updater/v2/CheckHistory`, the notice in
`MainWindow::updateCheckedV2`; not committed): after 14 days and at least 3 failed checks,
again every 14 days. Tried on a files2 build with a dead proxy and a history set 15 days
back. The "last checked" line in the settings is not done.

### N3. The serial can go back, and a missing tags document restarts the tree (G2, wider)

G2 is right: `lib/updater_tree.py:157` takes the serial from the tree's own tags document.
Three more ways to the same result:

- **A missing `tags.json` is a new tree.** `updater/deploy.py:95-97` then writes serial 1
  and makes the build being deployed Latest, Testing *and* Stable. The gate passes it (the
  new build is the highest, every tag names a present build). So if the local tree ever
  loses that one file, the next push to `develop` becomes Stable for everyone who can still
  read it, and every Launcher with a stored serial refuses it. That is fail-open on the
  two things that matter most.
- **`files/publish.py` says "restoring an earlier snapshot and publishing puts the live
  site back to it"**, and `deploy.py`'s own error messages point at `snapshots.py restore`.
  That was true for the current layout. For the new tree it publishes a lower serial.
- **files2 shows what a restart looks like.** Its tags document today (2026-11-11) has
  serial 17, Stable 5052 and Latest 5069, although the tree has existed since 5025: it was
  flushed on purpose and started again at 5052, because earlier updater bugs (fixed since)
  had already stranded the older builds there. Any Launcher from the old tree with a
  stored serial above 17 now refuses the new one. That is fine on files2, which is
  disposable; in production the same flush would cut off every install, and nothing in
  hopper stands in its way. It also means stranding by a bug has already happened once, on
  the test system, which is the case G1's notice and N5's update test are for.

Done in hopper 2026-11-11 (staged, not committed), all three:

- **The serial never starts low:** `max(tree's serial + 1, the current time in seconds)`.
  The format allows it ("increases by at least one"), integers go to 2^53, and a restored
  or restarted tree is then above everything published before without any state to keep.
  A build server clock that is wrong once only makes a jump.
- **A new tree only when asked:** `deploy.py --new-tree`, otherwise a missing tags
  document is an error; with it, a tree that has builds is refused.
- **`publish.py` compares serials** with the published snapshot's tags document and
  refuses a lower one (and a different document with the same one).

Left open: after a restore, only a deploy or a tag move writes a new tags document. There is
no command that just writes the same tags again with a new serial.

## Server-side mistakes that Launchers refuse silently

### N4. A directory that becomes a file can't be applied (tried)

`planInstall` removes first, then places. With `a/b` in the installed build and a file `a`
in the next one: the removal moves `a/b` to `update/backup/a/b`; the placement then backs
up what is at `a`, the now empty directory, to `update/backup/a`, which exists and isn't
empty. `rename` fails (tried with the same two renames: `ENOTEMPTY`), `Installer::backUp`
(`Install.cpp:187`) reports "couldn't move out of the way", the update rolls back, and it
does so on every attempt, for every Launcher on a build with the directory.
`test_replacesDirectory` covers a stray directory without a current manifest, not this.

The other direction (a file that becomes a directory) applies, but its rollback fails.

Rare in practice; a macOS framework or a `jars/` reshuffle could do it. Like G4 it is
between two builds, where no check looks (N5).

### G4. Case-only renames: confirmed, narrower than it sounds

Only when the content is unchanged. With changed content the removal runs first and the
placement finds nothing, so it works.

### N5. Nothing compares a build with the one before it, and nothing tests an update

The validator checks each manifest and the tree, never two builds against each other. What
slips through, each stranding the Launchers of one variant until a fixed build, each either
silent or one "Update not possible" message:

- a variant missing from a build that gets tagged (a platform left out of `PLATFORMS`), or
  a `supersedes` chain cut by dropping a middle variant (Updater.md already notes this one);
- a requirement key added to an existing variant: every installed Launcher of that variant
  counts it as unmet, its own variant included;
- N4 and G4.

Two things, the second covering the first:

- **A continuity check in `deploy.py` and `tag.py`:** against the tag's current build, every
  variant is still there or has a successor, its requirement keys are the same, no path
  changes only in case, no path is a file on one side and a directory on the other. Each
  with an explicit override for when it's meant.
- **An update test before a tag moves.** Install the tag's current build from the tree,
  point it at a local copy of the tree with the candidate, update, start, check again and
  expect "no update". This runs the old Launcher's real code against the new build, and the
  new build's updater once, so it also catches G6 and G7 for that platform. Linux can run
  in the build containers. It needs a way to update without the dialogs: a command line
  switch that checks, applies and exits with a status, which doesn't exist yet. (hopper's
  `launcher/gui.py` is a helper for looking at the GUI by hand, not something to build a
  release gate on.) Latest could require it for the previous Latest, Stable for the
  current Stable.

Done 2026-11-12/13. The continuity check is in `updater/deploy.py` and `updater/tag.py`
(refusing a dropped variant, a new or stricter requirement; case-only and file/directory
path changes are warnings, since the Launcher handles them now), with retirements declared
in hopper's platform table. The update test became "the way back to Stable": each new
Latest, headless (`--update`), goes to Stable and back up against the local tree before it
is published (hopper's `updater/update-test.py`, in the release job). It runs Latest's own
updater, so a build whose updater is broken is caught before it ships, which testing from
the previous build couldn't do.

### N6. Every check needs a real 404

The keys walk asks for `keys/<N+1>.json.sig` on every check and only ends when a mirror
answers 404 (`Checker.cpp:168`; `wasNotFound()` is `ContentNotFoundError`, HTTP 404 and
nothing else, `net/NetAction.h:95`). A host that answers a missing file with 403, 410, a
redirect or an error page with 200 fails every check of every Launcher, silently. One
mirror with a real 404 is enough.

- Tried 2026-11-11: both files2 hosts answer 404 for `updater/keys/2.json` and its
  signature. The bucket's 404 is an HTML page, which is fine.
- Worth a line in UpdaterFormats.md (Serving), and a probe after each publish: the next
  keys version must be a 404 on every base URL.
- A correction to the first review (3.2): the walk ends when *any* mirror said 404 and the
  others failed in whatever way, not when every mirror said it.
- Cost, not a strand: the 404 from the files server doesn't end it, the bucket is asked
  too. So every check of every Launcher makes one request to the bucket, which the mirror
  order was meant to avoid.

## Smaller

- **N7. The stored state doesn't say which tree it belongs to.** `State::load` takes a
  stored keys document with a higher version and a stored serial as they are. A data
  directory that has seen another setup (a files2 build, then a production one) keeps the
  other tree's keys or serial and fails every check until `updater/state.json` is deleted.
  Testers only. Storing the pinned root keys in the state and dropping it when they differ
  would end it.
- **N8. The shadow tree's first build becomes Stable.** With the tree turned on in
  production before the bridge, its first build is a v1 build, tagged Stable and archived
  for good. And the README's ceremony step sets `UPDATER_ROOT_KEYS` for production while
  `UPDATER_VERSION` is still 1, which `common.updater_floor` refuses (`lib/common.py:469`).
  With N1's order the tree's first build has updater 2 and the question goes away;
  otherwise decide whether the tree is wiped before the bridge (see N3 for the serial).
- **N9. Staging directories pile up.** `update/XXXXXX` is kept when a download or an apply
  fails, a full copy of the changed files each time.
- **G3.** Agreed. The 90-day warning is a log line in a release job; it should be something
  that can't be missed (the hopper UI, a failing job from 30 days).
- **G5.** Agreed, and the worst case is the first variant move: the Windows Qt 5.6 to 5.15
  switch replaces every Qt DLL in one apply. Still a window of milliseconds per file.
  Accepted 2026-11-12 (Updater.md, "Interrupted installs are accepted"): recovery code would
  run in the binary that the interruption may have broken; revisit with a separate updater
  process.
- **G7.** Agreed. `UPDATER_ENABLED` also needs the refspec to start with `refs/heads/`
  (`BuildConfig.cpp.in`), which hopper gets by checking out `-B <branch>`; the gate only
  looks for the keys. N5's update test covers it where it runs; for the rest, the gate could
  look for the variant ID in the binary as it does for the keys.
- **G8.** Agreed, and Windows has the same shape: a Launcher run in compatibility mode
  sees an older Windows. Both only keep an install on its variant.
  Fixed 2026-11-12: Windows takes the version from the kernel's shared data page, then
  ntdll's `RtlGetNtVersionNumbers` (both real in XP, 7 and 8 modes on Windows 11; the
  registry's CurrentBuildNumber is faked too); macOS from `kern.osproductversion` (not tried
  on a Mac; today's builds use SDK 11.1, which doesn't get 10.16 anyway).
- **G9, G12.** Agreed as written.
- **G11.** Tried 2026-11-11: on both files2 hosts `tags.json`, its signature and
  `mirrors.json` come with `Cache-Control: no-store` and aren't served from the cache. To
  repeat on production's hosts once the tree is there.

## What looked right

- Signatures over the raw bytes before parsing, a document and its signature from the same
  mirror, the type and number checked against the file asked for.
- Compiled-in hosts can't be removed by a mirror list; a lagging mirror is skipped, not
  trusted.
- The publish order, removals last, the tags document after everything it names; a failed
  publish leaves the old pointers.
- Stable builds and tagged builds are never purged, and a Launcher on a purged or withdrawn
  build still updates.
- A broken or lost `state.json` falls back to the floors, part by part.
- Variants a Launcher can't read are left out instead of failing the build.

## The list

| # | What | Reach | Silent? | Before the bridge? |
|---|---|---|---|---|
| N1 | First updater 2 build becomes the bridge automatically, untested in production | everyone, for good | yes | yes |
| G1 | No notice when checks keep failing | makes the rest silent | — | yes, in the bridge |
| N3 | Serial goes back: restore, restart, missing tags document (also makes the build Stable) | everyone, until the serial catches up | yes | yes |
| N5 | No check between builds, no update test before a tag moves | a variant, until a fixed build | mostly | the update test, yes |
| N4 | Directory becomes a file: can't be applied | everyone on the older build | no (error each time) | with N5 |
| N6 | Checks depend on a real 404 for the next keys document | everyone, while it lasts | yes | written down and probed |
| N7 | Stored state from another tree | testers | yes | no |
| N8 | Shadow tree's first build is a v1 build and Stable | tidiness, runbook | — | decided with N1 |
| N9 | Staging directories pile up | disk space | — | no |
