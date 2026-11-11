# Linux packaging

How the Launcher gets onto Linux machines: the wrapper packages, what they become, and how
that grows into [abipak](https://github.com/MultiMC/abipak), a separate project. Notes from
2026-11-09.

abipak comes in stages (its README): first a minimal helper for the Launcher's move to
Qt 6 on Linux (One requirement format, below), being worked on, including telling users
about missing Qt 6 libraries and offering to install them on tested distributions
([Updater.md](Updater.md), Qt 6 variants); then a wrapper replacing `run.sh` (Step 2 here);
then, maybe, a support layer distributions install and integrate themselves. The wrapper
packages' refresh (Step 1) doesn't wait for any of it.

## The wrapper packages today

Three packages, all installing the same small wrapper:

- **Debian, Ubuntu:** the Packaging repo's `ubuntu/` (built by hand with `dpkg-deb`; the
  published one is `downloads/multimc_1.6-1.deb` on the files site, linked from the
  website, while the repo's `control` says `1.6-2`).
- **Fedora, RHEL:** the Packaging repo's `rpm/`, built from the same files.
- **Arch:** the multimc-pkgbuild repo, on the AUR, which repackages the published `.deb`
  (checked with `sha1sums`).

They install a desktop file, the metainfo, an icon, the man page and `opt/multimc/run.sh`.
On first start `run.sh` downloads `downloads/mmc-stable-lin64.tar.gz` (or lin32), the
current layout's Stable package frozen in 2023, with `wget` and a `zenity` progress
window, and unpacks it into `~/.local/share/multimc`. The Launcher updates itself from
there. Later starts only run it.

What's wrong with it:

- the first download isn't verified (only TLS), unlike everything the updater fetches;
- new installations start from a 2023 build, and depend on the current layout for good;
- the dependencies are written by hand in three places, and drift (`zenity` and `wget` only
  for the download; `openjdk-8-jre` recommended long after it stopped mattering);
- three packages built three ways, none of it reproducible or checked.

## Step 1: refresh them

Keep the design, fix what's there:

- one place builds all three from the same files, and the versions agree;
- drop what no longer matters (`openjdk-8-jre`);
- at the updater cutover, `run.sh` downloads the new layout's Stable package instead
  ([Updater.md](Updater.md), The cutover: the wrapper packages). The frozen 2023 packages
  stay, for wrappers already out there.

## Step 2: a small tool instead of run.sh

abipak's second stage, after the Qt 6 helper: a static binary that

1. **probes** the host for the Launcher's requirements: finds each library the way the
   dynamic loader would and reads the symbol versions it defines, and says what's missing
   ("needs glibc 2.17, this system has 2.12", "no Qt 5 Widgets library");
2. **fetches and deploys** the Launcher: from the updater's tree, verified the way the
   Launcher's updater verifies (signed documents, content files by SHA-256, the mirror
   list), so the very first download is signed too; picks the variant whose requirements
   the host meets, as the updater does;
3. **runs** it.

It's shipped the way an AppImage feels to use (abipak's README, How it feels to use): one
file, `MultiMC-x86_64`, made executable and run. The first run does the work, later runs
start the Launcher straight away. The file's name decides where the Launcher and its data
go: a name with the word "portable" in it (the website can offer `MultiMC-portable-x86_64`,
the same bytes) keeps them next to the file, as MultiMC has always been portable; otherwise
`~/.local/share/multimc`, where the wrapper puts them today, so those installations carry
on. An installation already next to the file always wins over the name.

The packages can keep declaring their dependencies and shrink to installing this one file
and a desktop entry; the tool explains what's missing when the dependencies aren't enough.
The tarball on the website can become this file too.

## Step 3: resolve and install

Once probing has proven itself (the start statistics, [Statistics.md](Statistics.md), show
which distributions matter): turn what's missing into distribution packages and install
them with the user's consent. How distributions answer "which package has this ABI" is in
abipak's README (Stage 2). We integrate with the distributions that matter; elsewhere the
tool only explains what's missing.

## One requirement format

The Launcher's build pipeline already measures what each build needs (the compatibility
gate), and the updater checks it before offering a variant (`requires` in the build
manifest, [UpdaterFormats.md](UpdaterFormats.md)). Moving to Qt 6 on Linux needs a richer
requirement anyway: a list of libraries with minimum symbol version nodes
([Updater.md](Updater.md), System requirements, Qt 6 variants). That list is exactly what
abipak's measure writes and its probe checks, so it should be one format: what the gate
measures, the updater checks before offering a variant, and the tool checks before
installing.

abipak's prototype (2026-11-10, its README, The prototype) settles the shape: per library,
its soname and every symbol version node the binaries need from it, exactly what the
dynamic loader checks:

    "libs": [
      {"soname": "libQt5Core.so.5", "versions": ["Qt_5", "Qt_5.6"]},
      {"soname": "libstdc++.so.6", "versions": ["CXXABI_1.3", "GLIBCXX_3.4", ..., "GLIBCXX_3.4.21"]},
      {"soname": "libz.so.1", "versions": []}
    ]

Measured from build 5055's lin64 package, that's 12 libraries, carrying the same glibc 2.14,
GLIBCXX 3.4.21 and Qt 5.6 floors the gate reports today, but per library and exact. Checking
it needs no knowledge of how each library numbers its versions, and the same binary gave
the right answer on Debian 12, Ubuntu 22.04, Fedora 41 and Arch-based CachyOS, including
"Qt 6.4 lacks Qt_6.8" for a Qt 6.8 requirement.

Proposed for the build manifest: a Linux variant's `requires` with `os`, `arch`, the
`interpreter` and `libs`, in place of `glibc`, `glibcxx` and `qt` (Launchers that can't read
it leave the variant out, [UpdaterFormats.md](UpdaterFormats.md), Requirements). Next, in
order: the compatibility gate writes `libs` into the manifest's `requires`; the Launcher's
updater checks it the way probe does (a port of probe to C++, or abipak's static binary
shipped in the Linux builds and run as a child process); then the Linux Qt 6 build and its
variant.

## Open questions

- Where the packages are built: hopper (reproducible and checked like the releases) or by
  hand as now.
- Whether the tool goes into the existing packages (replacing `run.sh`) or ships on its own
  too, as a download on the website.
