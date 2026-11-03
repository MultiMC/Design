# Meta format v2

An overhaul of the metadata format the Launcher reads from meta: the package indexes and
version files that describe Minecraft, the mod loaders and their libraries. Started as notes
on 2026-11-03; nothing is decided yet. If the format changes this much anyway, almost anything
is on the table, including how it is published and cached.

Related: [Runtimes.md](Runtimes.md) (Java runtimes), [LauncherNetwork.md](LauncherNetwork.md)
(sites, mirrors, caching).

## What's wrong with v1

Checked in meta's generators and the Launcher source on 2026-11-03.

- **No Java requirements.** Mojang's version files say which Java runtime they need
  (`javaVersion`: a piston component and a major version). Meta reads it and drops it. The
  Launcher instead checks hardcoded release dates (`VerifyJavaInstall`: Java 8 from 17w13a, 16
  from 21w19a, 17 from 1.18 Pre Release 2, 21 from 24w14a, 25 from 26.1 Snapshot 1), and knows
  nothing about mod loaders' own requirements.
- **Arguments are mostly stripped.** Since 1.13, Mojang's `arguments` has game and JVM arguments
  with rules and features. Meta flattens the game arguments into the old
  `minecraftArguments` string, drops anything with rules, and removes `--clientId` and
  `--xuid`. The JVM arguments are ignored, except `-XX:StackShadowPages`, which becomes a
  trait; the Launcher builds the JVM command line itself. For legacy versions, meta removes the
  game arguments too, and the Launcher uses hardcoded ones.
- **Loose parsing.** `formatVersion` 0 and 1 are both accepted, optional fields fall back to
  defaults, and unknown fields are ignored. A broken or misunderstood file can load as
  something else instead of failing.
- **Hashes unused.** The root index lists each package index's SHA-256, and each package index
  each version's. They match the bytes served, but the Launcher never checks them.
- **Whole-file package indexes.** Each package index lists every version. Forge's is 1.8 MB
  with 5,092 versions, and it is downloaded again whenever a single build is added.

## Directions

Ideas so far, none of them decided.

- **Runtimes as dependencies.** Versions name the Java runtime they need: a range of major
  versions, and the piston component that provides it. Mod loaders can narrow the range. The
  Launcher resolves the range across an instance's components and picks or installs a runtime
  ([Runtimes.md](Runtimes.md)).
- **Arguments kept whole.** Carry Mojang's structure (rules, features, JVM and game arguments)
  instead of flattening it, and evaluate it in the Launcher, which then knows why each
  argument is there.
- **A strict format.** One format version, exact keys, bounded numbers, and fixed patterns, read
  with the Launcher's strict JSON readers. Anything not understood is refused, so a newer
  format fails clearly in an older Launcher.
- **Published by hash.** Next to the paths, every file is also published at its SHA-256. Those
  files never change, so they can be cached forever at the edge with no purge, and only the
  root needs one. Today meta can't be edge-cached at all, because nothing purges it.
- **Cached by hash.** The Launcher keeps files by their SHA-256 and the last good root. Only the
  root is fetched to see what changed. Files it already has cost no request, and every
  download is checked against its hash ([LauncherNetwork.md](LauncherNetwork.md), Caching).
- **Less repetition.** Version files repeat the same library lists almost word for word.
  Measured on meta's output on 2026-11-03 (9,096 files, every package and version):

  | Form | Size |
  |---|---|
  | as published (pretty-printed JSON) | 235 MB |
  | minified | 147 MB |
  | minified, zstd -19 --long, as one blob | 1.5 MB |

  As one blob, Forge is 930 KB, NeoForge 460 KB, Minecraft 150 KB and the rest about 110 KB.
  Meta changes several times a day, and a binary diff against the previous run is still about
  150 KB, so one blob doesn't suit frequent updates. Shared parts referenced by hash (library
  sets, for example), or package indexes split into pieces that don't change, may help more.
- **Side by side with v1.** Existing Launchers read `v1/`, so v1 keeps being published as it
  is, with v2 next to it.

## Open questions

- What a version file contains, and what is shared between versions.
- How the Launcher evaluates rules and features, and which of them it supports.
- How runtimes are described: in version files, or in a package of their own.
- Whether the root is signed, like the new updater's documents.
- How the package indexes are split, if at all.
