# The Launcher's network layer

How the Launcher downloads from MultiMC's own servers, and how it falls back to other copies
of them. It started with the updater overhaul ([Updater.md](Updater.md)), but it applies to
everything the Launcher fetches from us, not only updates.

## How it works today

Checked in the Launcher source (develop) on 2026-11-02.

- **One URL per endpoint.** Every place the Launcher fetches from us has one compiled-in URL:
  `META_URL`, `TRANSLATIONS_BASE_URL`, `NOTIFICATION_URL`, `UPDATER_BASE`, `NEWS_RSS_URL`
  (CMake options per setup) and `FMLLIBS_BASE_URL` (fixed in `BuildConfig.h`). A test setup
  such as files2 has to point each of them at its own host separately.
- **Downloads.** `Net::Download` writes into a sink (memory, a file, or the HTTP meta cache)
  and runs validators over the data as it arrives (for example `ChecksumValidator`).
  `NetJob` runs many downloads and retries a failed one up to three times, always on the
  same URL.
- **Second copies exist, unused.** The files tree and meta both have a copy in an R2 bucket,
  in sync with the main server: `files-r2.multimc.org` and `meta-r2.multimc.org` (and
  `files2-r2.multimc.org` for files2). They can't take over the main hostnames when a server
  is down: Cloudflare won't proxy a CNAME onto an R2 custom domain (see Updater.md, Mirrors
  and fallback in the Launcher). Only a Launcher that knows both names can use them.

## Sites and endpoints

Decided 2026-11-02.

- **A site** is one of our hosts, with the copies it has, in order of preference:

  | Site | Copies |
  |---|---|
  | files | `files.multimc.org`, then `files-r2.multimc.org` |
  | meta | `meta.multimc.org`, then `meta-r2.multimc.org` |
  | website | `multimc.org` only |

  The website has one copy, and that is enough: if it is unreachable, the news can wait.
- **An endpoint** is a site and a path on it: translations are files + `translations/`,
  notifications files + `notifications.json`, the FML libraries files + `fmllibs/`, the new
  updater's tree files + `updater/`, meta's tree meta + `v1/`, the news website + `rss.xml`.
- **Per setup.** CMake sets each site's base URLs (an ordered list) per setup, and they end
  up in BuildConfig like every other value. Endpoints only name their paths. A files2 build
  then sets the files site to files2 and its copy, and every files endpoint follows.
- **Only our own hosts.** Third-party services (Mojang, Microsoft, Modrinth, CurseForge, ...)
  keep their URLs as they are; there is nothing to mirror.

## Mirrors

Decided 2026-11-02.

- **In order.** A download goes to the site's first base URL. Only when that fails does it
  try the next one. The main server comes first because requests to an R2 bucket cost
  money; the copies are there for when the main server is down.
- **Any failure moves on**: unreachable, an HTTP error, too much data, a stalled transfer,
  or a validator that refuses the content (a wrong hash, a signature that doesn't verify).
  A missing file (404) moves on too: a copy that lags behind a sync may not have it yet.
- **Per request.** Each request starts again from the first base URL, so the main server is
  used whenever it is up. A base URL that couldn't be reached at all is skipped for a few
  minutes, so an outage doesn't cost a timeout on every request.
- **No need to stay on one copy.** Everything below a site's root is checked against
  something already trusted: meta's files against the hashes in the indexes above them, the
  updater's documents against their signatures and its files against their hashes. A copy
  that is behind can only fail, never hand back something inconsistent, and a failure moves
  on. Staying with one copy for a whole operation is therefore not needed for correctness.
- **Trust doesn't change.** The copies are ours, so content that isn't signed (meta,
  translations, notifications) is trusted from a copy exactly as from the main server today.
  Mirrors change availability, not trust.
- **Where the lists come from.** Compiled in, per setup. For the files site, the new
  updater's signed mirror list can replace the compiled-in one at runtime (see
  [UpdaterFormats.md](UpdaterFormats.md), The mirror list): it names the files site's roots,
  so every files endpoint follows it, signed or not. Other sites stay compiled-in only. Meta
  could join the signed list later; nothing needs it yet.

## Caching

- **Updates aren't cached.** They are downloaded, checked, applied, and then of no further
  use. The updater's downloads bypass every cache.
- **Meta should be cached by hash** (noted 2026-11-02). Its files form a hash tree: the root
  index lists every package with a SHA-256, and each package index lists its versions with
  one. Only the root needs to be fetched to see what changed; everything else can be looked
  up by hash, with no request at all when it is already stored, and the same hash holds on
  every copy. Today meta goes through the HTTP meta cache and ETags instead. Moving it to a
  store keyed by hash (possibly a key-value store rather than many small files) is a task
  of its own, after the sites exist.
- **The HTTP meta cache stays** for what it is mainly for: bigger downloads from external
  sources. Its entries are keyed by a base name and a path, not by host.

## Groundwork

What the network layer needs before the updater's new client is built on it (noted
2026-11-02). Each part lands on its own, compiled in every build, with unit tests, and
changes nothing for users until something uses it:

1. **`Net::Download` extensions**: a size limit (a validator that refuses more than a given
   number of bytes, as the data arrives), an option to bypass caches, telling a missing
   file (404) apart from other failures, and a timeout for a transfer that stalls.
2. **Sites and endpoints** as above: the base URLs in BuildConfig, downloads by endpoint and
   path, fallback per request. Then meta, translations, notifications, the FML libraries and
   the news move onto them. The current updater stays as it is: its layout is frozen at the
   cutover and keeps its absolute URLs.
3. **Meta's cache by hash**, later and separately.
