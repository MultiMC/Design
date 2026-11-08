# Start statistics

A replacement for the Launcher's old analytics: counts of application starts by system, so
decisions like dropping a platform rest on numbers. Proposal, 2026-11-08; nothing here is
built yet.

## Why

Platforms and variants are only retired with an idea of who is still on them. The first
question is Windows XP: if hardly anyone starts MultiMC on XP any more, the Qt 5.6 build can
go, and with it most of the cost of building with Qt 5 and Qt 6 at once (see
[Updater.md](Updater.md), System requirements, Qt 6 variants). The same numbers would show
which Linux systems have which Qt 6, before a Qt 6 floor is picked.

## What was there before

The Launcher had Google Analytics (Universal Analytics, through the `ganalytics` library):
a persistent client ID per installation, an opt-out page in the setup wizard, anonymized
IPs, and the system, version and Linux distribution as custom dimensions. It was disabled
in September 2023 when Google ended Universal Analytics, and removed from the code since.

## What is counted

Starts, not users or installations. Every start of the Launcher sends one small report; the
counts are reports per day. Someone who starts MultiMC ten times counts ten times, which is
fine for the question asked: how much use each system gets.

Because nothing has to be connected across starts, the report carries no identifier at
all: no client ID, no installation ID, nothing stored to recognize the same machine later.

## Principles

- **Nothing identifying.** No identifiers, no account or instance data, no paths. Only
  values that thousands of machines share.
- **Few fields.** Each one has to answer a question about platforms; nothing is collected in
  case it's useful later.
- **Visible and switchable.** On by default, with an opt-out (decided 2026-11-08): a setting,
  explained in the setup wizard with what is sent and a link to the code, and switchable
  later in the settings. It's a new setting, not the old `Analytics` one: this is different
  in nature (no identifier, starts only), so everyone gets asked afresh (decided 2026-11-08).
- **Never in the way.** Sent after the main window is up, without waiting for an answer;
  failures are ignored and never shown.
- **Inspectable,** like the rest of the release tooling ([Updater.md](Updater.md),
  Principles): every step can be run on its own, the counts are plain versioned files, and
  the raw data is gone once counted.

## The report

One HTTP GET per start to a static path on the files site, with the values in the query:

```
<files site>/stats/start?v=1&build=5052&variant=windows-x86-qt5.6&tier=latest&os=windows&os_version=5.1&cpu=x86
```

- `v`: the report's format, 1.
- `build`, `variant`, `tier`: the Launcher's build number, variant ID and update tier.
- `os`, `os_version`, `cpu`: from `SystemRequirements::detect()`, which the updater already
  uses: the operating system, its version (on Windows the NT version: `5.1` is XP), and the
  machine's CPU architecture (an x86 build on an x86_64 machine reports x86_64).
- On Linux also `glibc`, `qt` (the system Qt the Launcher runs on), and `distro` and
  `distro_version` (`ID` and `VERSION_ID` from os-release).

- `today=1` on the first start of a day (decided 2026-11-08), from the date of the last
  report kept locally (not an identifier: only a date). Counting only those gives machines
  active per day, still without telling machines apart, and keeps a few heavy users from
  skewing the numbers. Both counts are kept: all starts, and first starts of a day.

Only the first base URL of the files site gets it, not the mirrors: the counting reads one
server's log.

## Where it goes

The files server answers it like any static file (an empty file at `stats/start`), and its
web server logs it. That keeps the files server a static file server. It's a dedicated
server without much load, so the reports cost nothing noticeable there (decided
2026-11-08, rather than a Cloudflare Worker writing into Workers Analytics Engine, which
would be less inspectable).

- Cloudflare must not cache the path (a cache rule bypassing it, or the response's headers),
  or most reports never reach the server.
- The web server's log format for that path leaves out the client's address: the time and
  the query are all that's kept.

## Counting

A hopper job, daily:

1. Pulls the new log lines for `stats/start`.
2. Reads each query strictly: known fields only, each value matching its pattern (variant
   IDs as in [UpdaterFormats.md](UpdaterFormats.md), versions as dotted numbers). Anything
   else counts as invalid, by day, and is otherwise dropped.
3. Adds the day's counts by combination of fields to a versioned file per day.
4. Deletes the raw lines it counted.

hopper's web UI shows the counts: starts per day by OS and version, by variant, by CPU, and
the Linux breakdowns, each as all starts and as first starts of a day (`today=1`).

The numbers are indicative, not proof: anyone can send reports. That's accepted, since they
inform decisions rather than drive anything automatically.

## Using the numbers

The counts cover only builds that send reports, starting with the first one that does. XP
machines take the same develop builds, so they show up within days, but a decision about a
platform should wait for some weeks of data, since some people start MultiMC rarely.

## Open questions

None at the moment.
