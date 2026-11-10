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

The data layout move, binary diffs and fully automatic updates are not part of it.

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

Moving between variants (decided 2026-11-08, replacing the idea of 2026-11-06 that moves
might be optional): `supersedes` is the default, and users can opt out of it.

- **Moving is the default.** When a build has a successor of the installed variant that
  runs on the machine, the Launcher moves to it, but asks first. With a newer build, the
  update dialog has a ticked "Switch to <new>" checkbox; on the installed build (a successor
  added later), the Launcher offers the switch on its own, with "Switch now", "Keep <old>"
  and "Ask me later".
- **Staying.** Unticking the checkbox, or "Keep <old>", records the old variant in the
  Launcher's settings, and updates stay on it from then on.
- **The settings** (Updates page) list the variants of the tier's build that run on the
  machine, with the recommended one (where `supersedes` leads) first. Choosing another
  variant switches to it right away, on the installed build if needed. That is also how a
  user goes back to an older variant, as long as builds still have it.
- **Stranded variants.** When builds no longer have the variant a user stays on, the
  Launcher says so, even if they turned the move down before: the update dialog explains the
  move instead of offering to stay. If no variant runs on the machine any more, an automatic
  check says so once per build.
- **Names.** Variants have a `name` in the build manifest for this UI (UpdaterFormats.md,
  Variants); the Launcher never derives anything from variant IDs.

The first move is the Qt 5.6 Windows build to the Qt 5.15 one: `windows-x86-qt5.15`
supersedes `windows-x86-qt5.6`, so Windows 7 and newer move, and XP and Vista stay.

Later: Qt 6 variants (noted 2026-11-08, not planned yet). The move to Qt 6 should go through
the same mechanism.

- **Windows and macOS** bundle Qt, so a Qt 6 variant replaces the Qt 5 files like any other
  update. Chains already work: with Qt 6 superseding Qt 5.15 and Qt 5.15 superseding Qt 5.6,
  a Qt 5.6 install that can run Qt 6 goes straight to it.
- **Dropping a variant in the middle of a chain** cuts it: the successor map only comes from
  the variants in the build, so dropping Qt 5.15 while Qt 6 lists only Qt 5.15 leaves Qt 5.6
  installs without a successor. The successor has to take over the dropped variant's
  `supersedes` entries. hopper's validator could check a new build against the previous one
  for this.
- **Linux runs on the system's Qt**, and the `qt` requirement is checked against the Qt the
  Launcher runs on (`qVersion()`), so a Qt 5 Launcher never meets a Qt 6 requirement: safe,
  but nobody can move. The Qt 5 Launcher has to find out whether the system has a usable Qt 6
  without loading it into the Launcher's own process. Ways to do it: find the libraries the
  way the dynamic loader would and read their ELF symbol version definitions (as the
  compatibility gate measures glibc and GLIBCXX), or `fork()` and `dlopen()`/`dlsym()` in
  the child (the user's idea, 2026-11-08): a real load with `RTLD_NOW`, which also shows that
  every dependency resolves, and no extra binary to ship. The child touches no Qt, reports
  through a pipe and ends with `_exit()`; the parent gives it a timeout and kills it, and
  anything but a clear answer counts as not met (`dlopen` after `fork` in a process with
  threads isn't async-signal-safe, so a rare hang has to be survivable). Qt alone isn't enough
  to start (the xcb platform plugin's dependencies are separate packages), so this probably
  becomes a general requirement, a list of libraries with minimum versions, measured by the
  gate.
- **Which Qt 6 works:** any Qt 6 at least as new as the one the build is made against. Qt's
  public symbols are in one version node (`Qt_6`), next to empty marker nodes `Qt_6.0`,
  `Qt_6.1`, … that each include the one before; an application built against 6.N requires
  `Qt_6.N` (through `qt_version_tag`), so the loader refuses an older Qt. The Linux Qt 5
  build works the same way today (`MultiMC` requires `Qt_5.6`). So the compatibility gate can
  measure the minimum like glibc's, the highest `Qt_6.N` required, and a check on the user's
  machine only needs that node in the system's library (a `dlopen` fails without it).
- **ABI:** Qt keeps the public API binary compatible across 6.x; the private API
  (`Qt_6_PRIVATE_API`) changes with every minor release. The Launcher uses no private Qt
  headers, and the Qt 6 port must keep it that way, or the Linux build is tied to one exact
  Qt version.
- **Aim for a recent Qt 6** (2026-11-08): the Qt 5 Linux variant stays as the fallback, so
  machines without a new enough Qt 6 just aren't offered the move. Early 6.x releases
  weren't ABI stable, and even 6.6.2 removed a public symbol without changing any version
  node (`operator<<(QDebug, const QDockWidget &)`; Arch rebuilt 11 packages). Version nodes
  are enough for the check (decided): the floor goes above that break instead of checking
  every imported symbol, for example a later LTS such as 6.8.
- **The floor** is the highest of: that, the newest API the Launcher uses (checked by
  building against the floor in the toolchain image, as the Linux Qt 5 build pins 5.6), and
  the oldest distributions to reach. As of 2026-11-08, to be checked when deciding: Ubuntu
  22.04 has 6.2, Debian 12 and Ubuntu 24.04 6.4, Debian 13 6.8, Ubuntu 20.04 none. Newer APIs
  can still be used without raising the floor when they're reached at runtime instead of
  linked: following the system's dark mode needs `QStyleHints::colorScheme()` (6.5), which as
  a `Q_PROPERTY` can be read with `property("colorScheme")` and is simply missing before.
- **Both Qt versions in one codebase.** The Launcher then builds with Qt 5 and Qt 6 at once,
  and the expensive part is Qt 5.6 (the XP build), not Qt 5 as such: many of the Qt 6-ready
  replacements came late in Qt 5 (`Qt::SkipEmptyParts` and `Qt::endl` in 5.14,
  `QFontMetrics::horizontalAdvance` in 5.11, `QNetworkReply::errorOccurred` in 5.15). So a
  small compatibility header with wrappers chosen by `QT_VERSION`, in one place rather than
  `#if`s through the code, and one more toolchain image, Linux with Qt 6, built on every
  push. Once the XP build goes, the oldest Qt is 5.15 and most of that header can go too.
- **When the Qt 5 Linux variant is retired,** the users left on it (no new enough Qt 6) get
  the stranded variant notice.
- **Windows and macOS** use the bundled Qt, so their limit is the operating system Qt 6
  supports (Windows 10 and later, a recent macOS), in the variant's `os_version`.
- **Order:** a requirement old Launchers don't know isn't met for them (UpdaterFormats.md,
  Requirements). So first a Qt 5 build that can check the new requirement, and only once
  installs have it, a Qt 6 variant that supersedes Qt 5. Launchers that haven't updated stay
  on Qt 5, which builds still have.
- **Open:** a user who chose to stay on a variant stays on it, so they aren't offered a newer
  successor (Qt 6) either; whether a new successor should be offered again is to be decided.

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

### Fully automatic updates

Requested by a user, agreed 2026-11-08. Every push to `develop` makes a build, so Latest
users are asked to update often. An option lets the Launcher update without asking: it
fetches and applies the update of its tier on its own. It comes after the first release.

To work out then: whether it's on by default, when the update is applied (at startup or
exit, never while an instance is running), how the user learns what changed, and whether
the option applies to every tier or only some.

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
  trigger (on tagging Testing or Stable). The Linux wrapper packages come later (below).
- **The website goes last** (2026-11-09), when everything else is done and working: the
  bridge is Stable in the new layout and published in the current one.
- **The website offers Stable only** (2026-11-09), and one download per variant: both
  Windows variants, each with its minimum Windows version (XP and later; Windows 7 and
  later), not one that the updater then moves. The archive of earlier Stable versions is
  still to be designed.
- **The old download URLs** (2026-11-09): `downloads/mmc-develop-*` is linked from far more
  than the website (the wiki, guides, forum posts, other sites). At the cutover they are
  redirected (or otherwise pointed) to the new layout's Stable downloads.
- **The new download names** (2026-11-09, not settled): `MultiMC` instead of `mmc`, no
  branch name, the tier `stable` (the default), and the variant and/or platform, so the
  downloads can be told apart. Whether the variant ID or the platform is still open.
- **Telling users** (2026-11-09): an announcement at the cutover, written by Petr: what was
  done in the past weeks, what the updater changes, what follows next, and why almost
  nothing happened since 2022.
- **The wrapper packages** (noted 2026-11-08): the Debian package (the Packaging repo,
  `ubuntu/`), the RPM built from the same files (`rpm/`) and the Arch package
  (multimc-pkgbuild, which repackages the published `.deb`) all install the same
  `run.sh`. On first start it downloads `downloads/mmc-stable-lin64.tar.gz` (or lin32), the
  current layout's Stable frozen in 2023, and unpacks it; the Launcher then updates itself.
  Nothing forces them to change at the cutover: the frozen 2023 package updates through the
  current layout to the bridge build, which moves over. They change after the updater
  rework is in place (decided 2026-11-09), when `run.sh` (or what replaces it) downloads
  the new layout's Stable package instead, and new versions go out: a new `.deb` on the
  files site (and the website's link to it), the RPM, and the PKGBUILD (version, checksum,
  `.SRCINFO`) on the AUR. The URL `run.sh` downloads from may change later, at the cost of
  breaking old packages; they are a clunky old solution, and the aim is something better.
  When there are Qt 6 variants, the packages' dependencies (Qt 5 libraries today) need a
  look too. Where the wrappers go from here: [LinuxPackaging.md](LinuxPackaging.md).
- **Packages made by others** (2026-11-09): repackaging by others has been discouraged on
  purpose: it was usually mishandled and caused problems (sandboxing breaking things,
  patched Launchers with piracy enabled). Still, those platforms bring visibility, so the
  aim is better installation and delivery of our own (LinuxPackaging.md, abipak), not more
  third-party packages.

## Open questions

### The cutover (2026-11-09)

- **The new download names:** the variant ID or the platform (see The cutover above for
  what is decided).
- **The archive of earlier Stable versions** on the website: how it's presented, and
  whether the tag move writes an index for it.
- **If the bridge build turns out bad in production.** Everything after the bridge can be
  fixed with a new build in the new layout, as long as the bridge's own updater works.
  What can't: a bridge whose new updater fails in production (a platform where it can't
  apply, a configuration only production has). Launchers on it no longer read the
  current layout, and nothing in the new one reaches them; they need a reinstall by hand.
  How to rule that out before the freeze is open (being worked out).

### Version numbers (2026-11-09)

- **The "0.7.0" in the UI only confuses people.** Either versions mean something again, or
  the UI shows the build number (and tier) only.
- **How version and build numbers get into the binaries**, and that build numbers can't run
  out. Windows version resources hold four 16-bit fields (at most 65535 each); the
  Launcher's `branding/multimc.rc` has a fixed `1.0.0.0` today, so the build number isn't in
  them at all yet. macOS bundles get `major.minor.hotfix.build`; `BuildConfig` keeps it as
  an `int`. Builds are numbered around 5000 now, far from either limit, but this should be
  checked once, and decided before the build number goes into the Windows resources.

### The root keys

From planning the root key ceremony (2026-11-09; the steps are in hopper's README,
"Production's keys: the ceremony"):

- **Where the two root keys are kept**, each in its own place, and where the passphrases
  are kept, apart from them. Decided and recorded privately, not in this repo; the same
  goes for who else holds a copy (below).
- **The paper format:** the private key in plain (simple, but the paper must be guarded
  like the key) or the `age`-encrypted file (needs the passphrase too).
- **Who else can sign.** The keys document expires, and Launchers refuse an expired one.
  If nobody can use a root key before then, every Launcher stops updating. Two root keys
  cover losing one of them, not one person being unavailable; a key or a sealed copy with
  a second person would.
- **The first online key is made on the build server**, as files2's was, not offline: only
  its public key goes into the ceremony. (Rotation above says the ceremony makes it;
  probably the same change applies there.)
- **YubiKeys for the root keys** (tested 2026-11-10 with a YubiKey 5 NFC, firmware 5.7.4,
  throwaway keys in PIV slot 9c): one device per root key. The plan: make each key offline
  with a paper copy as above, then import it (`ykman piv keys import`); a PIV key can't be
  read back out, and the device records whether a key was generated on it or imported.
  Both kinds gave plain Ed25519 signatures over the raw bytes, which hopper's
  `keys/verify.py` accepts; the PIN is asked for on every signature. Signing worked through
  ykman's Python library (`yubikit`). openssl's PKCS#11 provider (pkcs11-provider 1.3.0)
  crashed on the freshly released OpenSSL 4.1.0, after making a correct signature; retry
  once that settles. Either way `keys/sign.py` needs a way to sign with a YubiKey slot.
  The second YubiKey is on its way.
- **Follow-up: keep a working copy of the YubiKey tools** (2026-11-10), as with the
  toolchains: a known-good, non-crashing set (`ykman`/`yubikit`, pcscd and the CCID driver,
  and whatever signs), kept and versioned, so the ceremony and later renewals don't depend
  on whatever a system update brings that week.
- **An idea, not decided:** a small Ed25519 tool of our own (Zig's `std.crypto`, or
  Monocypher as in the Launcher) in place of the `openssl` CLI in `keys/`: one static
  binary for the offline machine, no Python or openssl version to match.

## Related

- Build and deploy automation: the hopper repo (`README.md`; the files tree tools in
  `files/`).
- The files2 test system (`files2.multimc.org`, Cloudflare R2): a safe place to try a new
  updater end to end, `hopper/config/files2.env`.
