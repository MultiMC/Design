# Updater overhaul

Notes for the next version of the Launcher's updater: what is known and what has come up so
far, requirements and ideas alike.

The next updater gets its own repository layout, separate from the current one, so its
behaviour is free to change. Only the current layout has to keep working, for the
Launchers already out there, until they have moved over.

## Principles

No fully automated black boxes. For the updater, its release tooling and UI, and the build
automation behind them:

- Every individual part can be run on its own, and its intermediate results inspected.
- Automated pipelines fail predictably, and fail closed: on bad input or an unexpected
  format they stop and let nothing through to users, rather than being permissive.
- Intermediate results are versioned, so the real system can be debugged in place
  ("in prod"): run parts by hand, inspect, and undo any temporary mess afterwards.

The model is the meta pipeline, which does all of this: it refuses to publish anything when
something fails, it is strict about formats on purpose, and it versions its intermediate
results.

## How the current updater works

Checked in the Launcher source (develop 2a684511) on 2026-10-23.

- **Where it looks.** `Application.cpp` builds the channel list URL from two compiled-in
  values: `UPDATER_BASE` + platform + `/channels.json`. For example:
  `https://files.multimc.org/update/lin64/channels.json`.
- **Platform.** `getIdealPlatform()` (`Application.cpp`) can override the platform the
  Launcher was built for. On macOS it always returns `osx64-5.15.2`; that is how the old
  `osx64` Launchers were moved over. The Windows switch to `win32-5.15.2` is commented out
  ("5.15.2 is not stable on Windows").
- **Channel.** `UpdateChecker.cpp` always uses the `develop` channel, whatever the user
  picked. That is how `stable` users were moved over. `stable` still exists in
  `channels.json` but only very old Launchers read it (frozen since 2023-02-04).
- **Channel URL.** Each channel in `channels.json` has an absolute `url`. The Launcher
  follows it as is, so the files tree only works on the host it was written for.
- **Versions.** It fetches `<channel url>/index.json`, takes the version with the highest
  `Id`, and compares it with its own build number (`VERSION_BUILD`). Then it fetches
  `<channel url>/<id>.json` for the new and the current version and diffs the file lists.
- **Files.** Each file in a version has an MD5, permissions and absolute source URLs
  (`https://files.multimc.org/files/<md5>-<name>`). Files are downloaded, checked against
  the MD5 from the same manifest, and swapped in by the separate updater step.
- **Client caching.** The Launcher ignores HTTP cache lifetimes (`Cache-Control`, `max-age`)
  and decides by ETag and file hashes instead. So server-side cache headers only affect
  Cloudflare's edge, not Launchers.
- **Nothing is signed.** Whoever can serve the manifests (DNS, a certificate authority,
  Cloudflare, the origin, the bucket or its key) can ship code to every Launcher.
- **Past cutovers** (platform and channel moves) were done in this code and were always
  tricky. Launchers too old to have updated keep reading whatever their version reads.

## Scope of the first release

Decided 2026-11-01. The first release with the new updater (the one existing Launchers
update into) brings:

- tag-based releases and promotion;
- relative URLs and mirrors;
- signed manifests;
- system requirement checks before an update is offered or applied.

The data layout move and binary diffs are not part of it.

No version number marks it (2026-11-01). MultiMC is a stream of builds, and the version
scheme (0.7.0 in the Launcher's `CMakeLists.txt`) is vestigial: a build is identified as
MultiMC 5, its platform, and its build number.

## Cutover

Decided 2026-11-01.

One bridge build, published in the current layout, carries the new updater. The current
layout then freezes at that build: a Launcher still reading it lands on the bridge build
and moves over from there. After that, the unsigned path carries no new code.

The bridge build is tested on files2 first and then becomes the first Stable. Existing
installs go to Stable at the cutover. Since they are all on the bridge build, nobody is
ahead of Stable then.

## Ideas collected so far

### Tag-based releases and promotion

Every push to `develop` is a release today. Instead: `stable` and `develop` as tags or
pointers to builds. A build is made once and never rebuilt; promoting it to stable moves a
pointer to the exact build that was tested. The `stable` channel (and its download
packages) may come back this way.

Tagging means moving the tag:

- `develop` always points at the newest successful build (moved automatically).
- `stable` points at a build picked by hand (moved manually).
- Producing the download archives (`mmc-<tag>-<platform>`) is part of moving a tag, not
  of building. A build itself is channel-neutral and never changes afterwards.

Three tiers of stability (2026-10-24; the middle one named 2026-11-01):

- **Latest**: the newest successful build. This is today's `develop`; rename it to
  "Latest" in anything user-facing.
- **Testing**.
- **Stable**.

How builds move between them:

- An untagged build can be tagged Testing.
- A Testing build can be promoted to Stable.
- Builds that reach Stable are never purged.
- Once a build is tagged Stable, it goes into an archive of previous versions, which keeps
  its download archives (tarballs and zips). The old downloads are offered on the website,
  in an archive section.
- There is only one Testing build at a time.
- Testing resolves to Stable when there is no Testing build newer than the current Stable,
  for example right after a promotion (2026-11-01).
- The space used by the updater data is watched and capped at a configurable size, probably
  not counting Stable versions. Other builds are purged to stay under the cap, oldest first.
  Stable versions and the Testing build are never purged.

A build is one unit for all platforms (2026-11-01): one build number and one manifest that
lists each platform's files and requirements. A platform may be missing from a build. Tags
point at builds; the Launcher picks the entry for its platform.

Users can pick any of the three tiers in the Launcher (2026-11-01).

No rollback (2026-10-24): tags only move forward. A build is tagged Stable only after a long
enough period of testing that it doesn't cause breakage; a problem in a release is fixed
with a new build. Going back to an older version is not necessarily safe anyway: the
Launcher's data formats change, and an old version will not be made compatible with
formats from newer ones. Whether Launchers can install archived versions is decided later;
the archive keeps the data either way.

One exception (2026-11-01): when a user switches to a tier whose build is older than the
one installed (on Latest at 4310, switching to Stable at 4290), the Launcher offers a
downgrade, with a warning that newer data formats may not be readable. It only happens
when the user asks for it; otherwise a Launcher never goes to an older build.

Removing a build (2026-11-01): its files are deleted from the servers and any tag on it goes
back to the previous build. Launchers already on the removed build stay on it until the
next build; there is no forced downgrade.

Today's pipeline does it the other way: `release-build.py` names the packages after the
branch (`mmc-develop-*`) and `deploy.py` replaces the downloads with every build. In the
tag model, builds would be stored by build number, and the tag move would update the
channel and write that channel's downloads from the chosen build.

Translations template (2026-10-30): Laminar's `template-production` updates the template
for translators, queued today after every production Launcher release. With tags, it
should run when a build is tagged Testing or Stable instead.

### System requirements

Decided 2026-11-01. Each platform entry in a build's manifest declares what the build
needs:

- the operating system and its minimum version (for example Windows XP, macOS 10.13);
- the CPU architecture (x86, x86_64, and arm64 once there is a native macOS build);
- on Linux, the system libraries: minimum glibc and GLIBCXX, and the Qt 5 version, since
  the Linux builds run on the system's Qt.

The Launcher checks these before it offers or applies an update. If the machine doesn't
meet them, the Launcher stays on its current build and tells the user why (for example
"needs macOS 10.13"). This replaces the platform moves hard-coded in `getIdealPlatform()`:
one machine may run several variants of a build, and a variant can supersede another, which
moves Launchers over (see [UpdaterFormats.md](UpdaterFormats.md), Variants).

Idea (2026-11-06), not decided: `supersedes` moves people without asking, and that may not be
right for every move (a Windows machine can run both the Qt 5.6 and the Qt 5.15 build, and
some people will want one or the other). Moving could instead be optional and visible: when a
machine can run several variants of a build, the user chooses in the Launcher's UI. Until
this is settled, no variant supersedes another.

### Relative URLs in the manifests

The channel `url`s in `channels.json` and the source URLs in each version's file list are
absolute today. With relative URLs, a files tree works on any host unchanged: a test system
(files2), a mirror, a different bucket, promotion between channels.

Seen 2026-10-23: the files2 test system needed every `channels.json` rewritten, or a files2
build checked files.multimc.org for updates.

Note: the Launcher already resolves `index.json` and `<id>.json` relative to the channel
URL (`QUrl::resolved`), but not the channel URL itself relative to `channels.json`.

Existing Launchers only understand absolute URLs, so the current layout keeps them; it
freezes at the bridge build (see Cutover). The new layout uses relative URLs from the start.

### Signed manifests

Motivation: the recent case of an Indian certificate authority with weak private keys.
TLS and certificate authorities alone shouldn't decide what code every Launcher runs.

Decided 2026-11-01:

- **Keys.** Offline root keys are pinned in the Launcher. They sign an online key, which
  hopper keeps on the build server: a file only the hopper user can read, outside anything
  that is synced or deployed (never on the files server, in the bucket, or in a deploy).
  The online key signs every tag move. Keys are rotated through the root keys, so a lost
  or leaked online key isn't the end of updates (see Root keys and rotation).
- **What is signed.** The tag pointers and the build manifests, which then vouch for the
  files through their hashes.
- **Hashes.** SHA-256 instead of MD5 for the files.
- **Replay.** The signed pointers carry an increasing serial number, and a Launcher never
  accepts a lower serial than it has already seen. That stops anyone from rolling users
  back with an old signed pointer. It doesn't stop a freeze (an old pointer that is still
  the newest a Launcher has seen, served indefinitely to hold users back); that gap is
  accepted for the pointers. The keys document has an expiry (below), which limits how long
  a leaked online key can be kept in use that way.

#### Implementation

Decided 2026-11-01. Signing doesn't depend on any version of Qt (Qt 5.6 has no Ed25519):

- **Library.** The Launcher vendors Monocypher in `libraries/monocypher`, like `tomlc99`
  and `xz-embedded`: plain C99, no dependencies, no calls into the system for verifying,
  audited (Cure53, 2020), BSD-2-Clause or CC0. It uses the standard Ed25519 from the
  optional `monocypher-ed25519.c` (`crypto_ed25519_check`), not Monocypher's default
  EdDSA with BLAKE2b, so signatures match `openssl`, minisign and other tools.
- **Detached signatures over raw bytes.** Every signed file (the keys document, the
  pointers, the build manifests) has a detached signature over its exact bytes. The
  Launcher checks the signature first and only parses the JSON after that, so there is no
  canonical JSON to get right and nothing untrusted gets parsed.
- **SHA-256** for the files comes from Qt's `QCryptographicHash` (there since Qt 5.0); a
  hash is not a trust decision the way a signature check is.
- **hopper** signs with the `openssl` CLI (`openssl pkeyutl -sign -rawin`, Ed25519 key),
  the same tool as the offline ceremony; Python's standard library has no Ed25519.
- **Tests.** The Launcher's unit tests check that signatures made with `openssl` verify
  with Monocypher, and that a modified file or signature fails.

#### Root keys and rotation

Decided 2026-11-01.

- **Two root keys.** The Launcher pins two Ed25519 root keys and accepts a signature from
  either. They are made separately and kept in different places, so losing one key, or the
  place it is kept, doesn't end updates. With a single root key, losing it would leave
  reinstalling by hand as the only fix for every installed Launcher.
- **Made offline.** Each root key is generated on a machine without network (for example a
  live USB booted without networking), with a stock tool (`openssl genpkey -algorithm
  ed25519`, minisign or signify). Root keys never touch the build server or a workstation.
- **Stored twice.** Each root key exists as a passphrase-encrypted file on a USB stick (for
  example `age -p`) and on paper (the 32-byte private key as one line of text or a QR code,
  with a checksum). The two root keys are kept in two separate places; passphrases are
  kept apart from the sticks.
- **The keys document.** A small JSON file in the served tree, also kept in git: a version
  number, an expiry time, the current root public keys and the current online public
  key(s), signed by the root keys. The Launcher keeps the newest version it has verified,
  never accepts a lower one, and refuses an expired one. The expiry is long (about two
  years); a new version is signed before then, which also checks the ceremony and the
  backups regularly.
- **Rotating the online key** (routine, or after a leak): an offline ceremony makes the new
  online key, writes the keys document with the version raised and the old online key gone,
  and signs it with a root key. hopper gets the new key and publishes the document;
  Launchers drop the old key as soon as they see it.
- **Rotating a root key** (rare): the new keys document lists the new root keys and is
  signed by the current ones. The Launcher only accepts a change of roots through that
  chain, one version at a time from the one it knows, so a Launcher that missed several
  versions walks forward through them. New releases compile in the new root keys.
- **Scripted.** One script per step (make an online key, write the keys document, sign it,
  verify it), each runnable on its own. Only small text files go to and from the offline
  machine, on a stick.

Relative URLs and signatures fit together: the signature covers the content, not the
host, so any host can serve the same signed tree.

### Mirrors and fallback in the Launcher

The Launcher itself should fall back to another copy when the server is unavailable: the
bucket, or more generally any number of mirrors. Switching hosts by hand (DNS, Cloudflare)
is not good, and doesn't work for R2 anyway: Cloudflare won't proxy a CNAME onto an R2
custom domain (error 1014, tested 2026-10-24), so a bucket can only take over a hostname by
being connected to it in R2's settings.

Where the list comes from (2026-11-01): a few hosts are compiled into the Launcher to start
from; the signed data carries the current list of mirrors, so mirrors can change without a
release.

The compiled-in hosts (2026-11-01) are the two copies of the files we have: the files server
(`files.multimc.org`) first, then the R2 bucket (`files-r2.multimc.org`, live and in sync;
meta has one too, `meta-r2.multimc.org`). The entries are hostnames, not machines: if the
files server is ever replaced, the same DNS entries are pointed at the new one, and nothing
in the Launcher changes.

Mirrors are not the updater's own (decided 2026-11-02): the Launcher's network layer has
them for every site of ours (files, meta, the website), and every endpoint on a site uses
them, the updater's tree among them. The signed mirror list names the files site's roots,
not the updater tree's, so all of the files site follows it. See
[LauncherNetwork.md](LauncherNetwork.md).

This fits relative URLs and signed manifests: the Launcher can fetch the same signed tree
from whichever mirror answers, and the signature, not the host, decides what it trusts.

Until then (decided 2026-10-24): Amazon S3 is dropped, and the loss of redundancy that comes
with it is accepted for the time being.

### Serving the tree

Decided 2026-11-01. The served tree is plain static files: content-addressed files that never
change, plus small signed pointers that are written last. Any static host serves it: the
files server, an R2 bucket, any mirror.

The earlier idea of a snapshot-aware server API (btrfs snapshots synced to the server, a new
endpoint that serves per-snapshot views to new clients) is not pursued; the static layout
achieves the same. hopper's btrfs snapshots of its local files trees (`files/snapshots.py`)
stay what they are: local versioning, to inspect and undo changes.

### Binary diffs between releases

Eventually the updater may download binary diffs between releases instead of whole files
(noted 2026-10-25, not part of the first release). Decisions until then should keep it
possible: keep old releases' files available (build trees, release directories, the files
tree), and keep files identifiable by content (hashes), so a diff from any published version
to the next can be made and checked.

Reproducible builds matter here more than diffs (noted 2026-11-04,
[Launcher#5795](https://github.com/MultiMC/Launcher/issues/5795)). Releases are now always
clean builds, and the builds aren't deterministic: timestamps, and on Windows leftover
linker memory, end up in the binaries, libraries and jars. So files that didn't really
change get a new hash in every build, and updates carry them anyway. With content-addressed
files, a deterministic build is what keeps an update down to the files that actually
changed.

### A UI for releases

Moving tags (tagging a build Testing, promoting it to Stable) needs tooling with a UI, not
just a server-side layout.

Decided 2026-11-01: a web UI on the build server, reachable only from a private network,
behind Caddy with authentication (the same pattern as TeamCity and the Laminar UI). It runs
the same scripts that can be run by hand, one per operation (tag Testing, promote to Stable,
remove a build, sign), so every part still works on its own.

From experience: when a rollback was actually needed, it was always easier to fix and push
until the problem was gone. The updater had no rollback logic, and there was no tooling or
UI for it. TeamCity didn't fit either: running builds in containers with layered,
inherited properties became complicated and obtuse.

Removing builds altogether should be possible too: when something ends up in a build that
needs to go, take the build out of every channel and delete its files, not just move a tag
away from it.

Material a UI could build on: every release build already writes a `manifest.json`
(commit, build number, version, package hashes, compatibility gate results, releasable).

### The data layout

The updater rework comes with a rework of where the Launcher keeps its data (noted
2026-10-25). It is not part of the first release of the new updater and comes separately
(2026-11-01). The earlier thought of calling the two together 1.0 is dropped: builds get no
version number (see Scope of the first release).

The data layout today is portable: everything lives next to the Launcher, and on macOS
inside the `.app` bundle itself. That is why the macOS build isn't signed: signing seals the
bundle, and the Launcher writes its instances, settings and updates into it. It also clashes
with how macOS treats downloaded apps:

- App Translocation: a quarantined app started from where it was downloaded runs from a
  read-only, randomized path, so writing into the bundle only works once the user has moved
  the app (moving it with Finder clears the quarantine).
- A native arm64 build would need at least an ad-hoc signature (Apple Silicon runs no
  unsigned arm64 code; the linker signs by default). The x86_64 build under Rosetta has no
  such requirement. The updater would then have to replace each binary as a new file (write
  elsewhere, rename), never overwrite one in place: macOS caches the signature per file and
  can kill a process whose signed binary was modified.

Moving the data out of the program needs a careful migration: the existing installations
(instances, accounts, settings, icons) have to move to the new layout reliably, once, with
a way back if it fails part way, and without losing anything. It deserves its own design
and testing, on files2 first.

## Building it

Decided 2026-11-01: the new updater is built in the Launcher's `develop` branch, behind a
feature toggle. repoman stays as it is (it stops mattering after the cutover); its
replacement, which writes the new layout, is built separately.

Every push to `develop` is a release, so what this means in practice:

### Components

The work splits into parts that can mostly be built and introduced on their own (noted
2026-11-01):

1. **The formats.** The build manifest, the tag pointers, the keys document, the signature
   files and the tree layout, as a short spec in this repo, with a validator script in
   hopper that the publisher and the tests both use. Everything else reads or writes these,
   so this comes first. The spec: [UpdaterFormats.md](UpdaterFormats.md).
2. **Requirements from the compatibility gate** (hopper). `compat-check.py` already measures
   what each binary needs: minimum macOS, PE subsystem version, the highest GLIBC and
   GLIBCXX symbol versions, the Qt libraries. Recording the measured values in each
   release's `manifest.json` gives the new manifests their requirements from the binaries
   themselves, not written by hand. Changes nothing for users.
3. **Key tooling.** The offline scripts (make a key, write and sign the keys document,
   verify it) and test keys for files2. The production root keys are only needed before
   the bridge build, but the tooling can be finished and rehearsed early.
4. **The new publisher**, repoman's replacement (hopper). It reads a release and writes the
   new layout: content-addressed files, a manifest per build, signed pointers. It runs on
   files2 first, then in production as a shadow: a real new-layout tree on both hosts that
   no Launcher reads yet, so real publishes test it long before the cutover.
5. **Tag operations** (hopper). Tag Testing, promote to Stable, remove a build, retention,
   and writing the download archives on a tag move: one script each, on the tree from 4.
   They can run in shadow too.
6. **Launcher: signature checks.** Monocypher vendored, a function that checks a signature
   over bytes, and unit tests against signatures made with `openssl`. Nothing calls it yet,
   so it can land without the toggle, and it shows early that all four toolchains build it.
7. **Launcher: system detection.** Reading the OS version, CPU architecture, and glibc and
   Qt versions at runtime, and comparing them with a set of requirements. Useful on its own
   (the log, system information), so it lands without the toggle.
8. **Launcher: the new client**, behind the toggle, needing 1, 6 and 7. Relative URLs, the
   keys document and pointers, the stored state with its floors, the tier choice and the
   requirement checks. It has an apply step of its own (decided 2026-11-03, replacing reuse
   of v1's `UpdateController` and `OperationList`): it takes the target state of a build (the
   files and the symbolic links its manifest lists) and places it, with a backup and a
   rollback. Links (macOS frameworks need them) are made with `QFile::link`; a change to a
   link is applied, never refused. The current updater and its apply step stay exactly as
   they are and freeze with the current layout; whatever the two share is copied rather
   than changed, so the current updater needs no new testing. Users reach the new apply
   step only through the new updater. It is built on groundwork in the Launcher that lands first, each part on its
   own, without the toggle, and changing nothing for users (decided 2026-11-02):
   - **The network layer** ([LauncherNetwork.md](LauncherNetwork.md), Groundwork):
     `Net::Download` gains a size limit, a way to bypass caches, a missing file told apart
     from other failures, and a stall timeout; sites with mirrors and endpoints, with
     fallback per request, for every download from our servers.
   - **Strict JSON** in the Launcher's JSON helpers (`Json.h`): an exact set of keys, no
     nulls, integers in a range. The formats are read with these, not with code of their
     own.

   A first attempt at the client (2026-11-02) worked against files2, but was set aside: it
   had its own downloading and JSON parsing beside the Launcher's, its mirror logic only for
   the updater, `#ifdef`s for the toggle, and it refused updates that change links.
9. **The release web UI**, on top of the scripts from 5. Until it exists, the scripts are
   run by hand.
10. **The cutover**: the freeze in hopper, the website's downloads, the translations
    template trigger (see below).

Order: 1 first; then 2, 3, 6 and 7, which are small and independent of each other; then 4
and 5, running in shadow; then 8 against files2, its groundwork first; then 9; then the
rehearsal on files2 and the cutover. 4 and 5 are most of the server side, 8 most of the
Launcher work.

### In the Launcher

- **Updater v1 and v2.** The current updater is v1, the new one v2, each in its own
  directory and namespace (`launcher/updater/v1`, `UpdaterV1`; `launcher/updater/v2`,
  `UpdaterV2`; named 2026-11-03). They share no code: what v2 needs from v1 is copied, so v1
  stays exactly as it was until it is removed after the cutover.
- **The toggle is a CMake option**, `Launcher_UPDATER_VERSION`, 1 by default, not a runtime
  setting. It ends up in BuildConfig as a value, like the URLs, and that value decides which
  updater the Launcher creates (changed 2026-11-02). All code compiles in every build, with
  no `#ifdef` for it, so production builds contain the new updater, but nothing reaches it,
  and no hidden setting or broken config can move users onto an unfinished updater.
  Changing the option only rebuilds BuildConfig. hopper passes `Launcher_*` options per
  setup, so the files2 setup turns it on and files2 is where the new updater runs until the
  cutover. The compatibility gate checks the value, not the absence of the code.
- **Pinned keys are per setup**, like the sites' base URLs. files2 has its own test root keys. The
  compatibility gate checks that a production build pins exactly the production root keys,
  so a test key can never reach users.
- **Monocypher is linked statically**, so the gate's file lists don't change. It has to
  build with all four toolchains (g++ 5, MinGW 4.9.2 for XP, osxcross clang); the gate and
  the unit tests show that it does.
- **Updater state.** The Launcher stores the highest pointer serial and keys document
  version it has verified, next to the Launcher (the current data layout). Each build also
  has floors for both compiled in. If the stored state is lost or broken, the Launcher falls
  back to the floors and keeps updating; that only weakens the replay protection a little.
- **Settings.** The tier setting is new. It must not take over the existing update channel
  setting (ignored today), or `develop` users would land on Latest instead of Stable. The
  tier selector is only shown in builds with the toggle on.

### Before the cutover

- **Old Launchers take any different build number**, not only a higher one
  (`UpdateChecker.cpp` compares with `!=`). So in the current layout, removing the newest
  build sends every old Launcher back to the previous one. Once the layout freezes, the
  bridge build must keep the highest `Id` there for good and must never be removed.
- **The root keys are made first.** They are compiled into the bridge build, so the key
  ceremony comes before it, not at the end.
- **Both layouts in one tree.** The current layout stores files by MD5, the new one by
  SHA-256, so files exist twice while both are written. Only the new layout counts toward
  the retention cap; the frozen one doesn't grow.

### The cutover

- **Order.** The replacement publisher, signing and the online key are live in production,
  and the bridge build is Stable in the new layout, before the bridge build is published in
  the current layout. Otherwise the bridge's first update check finds nothing.
- **The freeze is enforced by hopper.** After the bridge build, the production deploy no
  longer runs repoman, and refuses to (a setting naming the build the current layout is
  frozen at), rather than just not doing it. Otherwise the next push to `develop` would
  quietly publish into the frozen layout again.
- **Rehearsed on files2 first, exactly:** install a files2 build with the toggle off, publish
  a bridge build with it on into files2's current layout, freeze that layout, and check that
  the installed Launcher moves over and lands on Stable. files2 numbers its builds
  separately, so production is not touched.
- **Changing at the same time:** the website's downloads (new installs get the Stable
  packages written when a tag moves, not `mmc-develop-*`) and the translations template
  trigger (on tagging Testing or Stable).

## Open questions

None at the moment.

## Related

- Build and deploy automation: the hopper repo (`README.md`; the files tree tools in
  `files/`).
- The files2 test system (`files2.multimc.org`, Cloudflare R2): a safe place to try a new
  updater end to end, `hopper/config/files2.env`.
