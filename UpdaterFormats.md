# Updater formats

The formats of the new updater's served tree: the tree layout, the build manifest, the tags
document, the keys document, the mirror list and the signature files. The publisher, the tag
operations and the Launcher's new client all read or write these, and hopper's validator
checks them. The reasons behind the design are in [Updater.md](Updater.md).

Decided 2026-11-01. This is format 1.

## Principles

- **Strict on purpose.** A reader refuses anything it doesn't fully understand: an unknown
  field, a missing field, a wrong type, a `format` other than 1, a value outside its
  allowed range. Nothing is skipped or guessed. A new field or a looser rule means a new
  format version.
- **Signature first.** Every document is checked against its signature before it is
  parsed. Readers never parse unsigned JSON from the tree.
- **No URLs in the documents.** Every path in the tree follows from the layout below,
  relative to the tree root, so the documents carry no URLs at all, relative or absolute.
  The only absolute URLs are the mirrors' base URLs in the mirror list.
- **Content never changes.** A content file, a build manifest and an old keys document are
  written once and never changed afterwards. Only the tags document and the mirror list are
  replaced.

## The tree

```
<root>/
  tags.json             the tags document
  tags.json.sig
  mirrors.json          the mirror list
  mirrors.json.sig
  keys/<version>.json   every keys document ever published, e.g. keys/1.json
  keys/<version>.json.sig
  builds/<build>.json   one manifest per build, e.g. builds/4312.json
  builds/<build>.json.sig
  files/<xx>/<sha256>   content files, e.g. files/9f/9f86d0...0a08
```

The root is a directory of its own on each host (for example `updater/` on the files server),
apart from the current layout's `update/` and `files/`. Where it is on each host is
configuration: the compiled-in hosts and the mirror list give the root's full URL.

- **Content files** are named by the SHA-256 of their content, as 64 lowercase hex digits,
  in a directory named by the first two of those digits. They hold the exact bytes of the
  installed file, uncompressed. A file that is the same in several builds or variants is
  stored once.
- **Build manifests** are named by the build number in decimal, without leading zeros.
- **Keys documents** are named by their version in decimal, without leading zeros. All
  versions stay in the tree, so a Launcher that missed several can walk forward through
  them.
- Every document has its signature next to it, with `.sig` appended to the name.

Nothing else is part of the tree. The download archives written on a tag move are not part
of the updater's tree and not covered here.

### Invariants

The validator checks these for a whole tree:

- Every content file a present manifest lists exists, with the size and SHA-256 the manifest
  gives. Content files no manifest lists are reported, not refused: a publish writes the
  files first, so an interrupted one leaves some behind.
- Every tag names a present build. No withdrawn build is present.
- `latest` is the highest present build number.
- The keys documents are numbered 1 to N without gaps, each one is signed as described under
  The keys document, and the newest has not expired. It warns when the newest expires within
  90 days.
- Every other document verifies with an online key of the newest keys document.

### Write order

The publisher writes so that a reader never finds a reference to something missing:

1. Content files.
2. The build manifest and its signature.
3. A new keys document or mirror list, when there is one.
4. The tags document and its signature, last.

Removing a build goes the other way: the tags document first (tags moved back, the build
added to `withdrawn`), then the manifest and its signature, then the content files no
remaining manifest lists.

Each file is replaced atomically (written under a temporary name and renamed, or a single
object upload). A document and its signature are still two files, so a reader can catch one
from before a write and one from after it, or a cache can hold them at different ages. The
signature check then fails, and the reader tries again later or tries another mirror; that
is the intended result. Hosts should serve `tags.json`, `mirrors.json` and their signatures
with a short cache lifetime.

## Common rules

These apply to every document (the manifests, the tags document, the keys documents and the
mirror list).

- **Encoding.** UTF-8 without a byte order mark. The whole file is one JSON object.
- **Fields.** Every field listed for a document is required, and no other field is allowed,
  at any level. `null` is never allowed.
- **Duplicate keys** are refused. The validator checks this before anything is signed;
  Qt's JSON parser can't detect them, so the Launcher relies on the signature for it.
- **`type` and `format`.** Every document starts with a `type` naming what it is (`build`,
  `tags`, `keys` or `mirrors`) and `"format": 1`. A reader refuses a document whose `type`
  doesn't match the file it fetched, so one signed document can't be passed off as another.
- **Integers** are JSON numbers without a fraction or exponent, from 0 to 2^53 − 1
  (9007199254740991), the range both Python and `QJsonValue` handle exactly. Build numbers,
  serials and versions start at 1.
- **Times** are UTC, written exactly as `YYYY-MM-DDTHH:MM:SSZ`, for example
  `2026-11-01T12:00:00Z`.
- **Hashes and keys** are lowercase hex: a SHA-256 is 64 digits, an Ed25519 public key
  (its raw 32 bytes) is 64 digits, a signature (its raw 64 bytes) is 128 digits. A key is
  identified by itself; there are no separate key IDs.
- **Version strings** (operating system and library versions) are numbers separated by
  dots, such as `10.13` or `3.4.21`, with no leading zeros except a lone `0`. They are
  compared numerically, part by part, a missing part counting as 0 (so `2.19` = `2.19.0` <
  `2.27`).
- **Sizes.** A reader refuses a document larger than 64 KiB, or 8 MiB for a build manifest,
  before checking its signature.
- **How the publisher writes them.** Keys sorted, two-space indent, ASCII only (anything else
  escaped), a final newline (Python's `json.dumps(obj, indent=2, sort_keys=True,
  ensure_ascii=True) + "\n"`). Readers don't depend on this, since the signature covers
  the bytes as they are, but it makes the output reproducible and easy to diff.

## Signature files

Every document has a detached signature file next to it: `<name>.sig`. It is ASCII text,
one signature per line:

```
<public key, 64 hex digits> <signature, 128 hex digits>
```

- Each line is the public key, one space, the signature, and a newline (LF). No blank
  lines, no other content, one to eight lines, no key twice.
- The signature is plain Ed25519 (RFC 8032, not the prehashed variant) over the exact bytes
  of the document. It is what `openssl pkeyutl -sign -rawin` writes and Monocypher's
  `crypto_ed25519_check` checks, as hex.
- A signature file is valid for a document when at least one line has a key that is trusted
  for that document and its signature verifies. Lines with keys that are not trusted for
  that document are ignored, so a file can carry signatures for a rotation overlap or from
  both root keys.
- A line whose key is trusted but whose signature doesn't verify makes the whole file
  invalid.

Which keys are trusted for which document:

| Document | Trusted keys |
|---|---|
| `keys/1.json` | its own `root_keys` |
| `keys/<N>.json`, N > 1 | the `root_keys` of `keys/<N-1>.json` |
| everything else | the `online_keys` of the newest verified, unexpired keys document |

## The keys document

```json
{
  "type": "keys",
  "format": 1,
  "version": 2,
  "expires": "2028-11-01T00:00:00Z",
  "root_keys": [
    "<64 hex digits>",
    "<64 hex digits>"
  ],
  "online_keys": [
    "<64 hex digits>"
  ]
}
```

- `version`: the document's version, the same number as in its file name. Each new
  document has the previous version plus one.
- `expires`: after this time the document is not used for checking anything except the next
  keys document. Normally about two years after it was signed.
- `root_keys`: one to four distinct keys, normally the two root keys. They sign the next
  keys document.
- `online_keys`: one to four distinct keys. They sign the tags document, the build manifests
  and the mirror list while this document is the newest. More than one only during a
  rotation.
- No key is in both lists.

A Launcher build embeds one keys document as its floor (for example `keys/3.json` as it was
when the build was made) and trusts it without a signature: it is part of the signed
binary's own contents, the way the root keys are pinned. The compatibility gate checks that a
production build embeds a production keys document.

How a Launcher uses them:

- It starts from the newest keys document it has stored, or the embedded floor if that is
  newer or nothing usable is stored.
- It fetches `keys/<known + 1>.json` and checks it against the known document's `root_keys`,
  and so on, one version at a time, until the next version isn't there. The intermediate
  documents may have expired; only the newest one has to be valid now.
- It never accepts a version lower than the one it has stored.
- If the newest keys document it can get has expired, it does not update, and tells the user
  why.

## The tags document

```json
{
  "type": "tags",
  "format": 1,
  "serial": 1042,
  "issued": "2026-11-01T12:00:00Z",
  "tags": {
    "latest": 4312,
    "testing": 4305,
    "stable": 4290
  },
  "withdrawn": [4311]
}
```

One document holds all three tags, so there is one serial to store and one file to write.
Every tag move, publish or removal rewrites and re-signs the whole document.

- `serial`: increases by at least one with every write. The Launcher stores the highest
  serial it has verified (falling back to its compiled-in floor) and refuses a lower one. An
  equal serial is the document it has already seen.
- `issued`: when the document was written. For people and logs; the Launcher doesn't make
  trust decisions on it.
- `tags`: exactly `latest`, `testing` and `stable`, each a build number. `stable` ≤
  `testing` ≤ `latest`. The publisher writes Testing already resolved: when there is no
  Testing build newer than Stable, `testing` is the same build as `stable`. A tag names a
  whole build; when that build has no entry for a Launcher's variant (see below), that
  Launcher gets no update from it.
- `withdrawn`: the numbers of removed builds, ascending, no duplicates, none of them a tag.
  Build numbers are never reused, so an entry stays for good. A Launcher on a withdrawn build
  can tell the user so; it still stays on that build until the next one (no forced
  downgrade).

On a fresh tree, the first build is all three tags, as the bridge build will be in
production.

## The build manifest

One per build, for all variants.

```json
{
  "type": "build",
  "format": 1,
  "build": 4312,
  "commit": "<40 hex digits>",
  "built": "2026-11-01T11:42:07Z",
  "variants": {
    "linux-x86_64-qt5": {
      "supersedes": [],
      "requires": {
        "os": "linux",
        "arch": "x86_64",
        "glibc": "2.19",
        "glibcxx": "3.4.21",
        "qt": "5.6"
      },
      "files": [
        {
          "path": "bin/MultiMC",
          "sha256": "<64 hex digits>",
          "size": 2803712,
          "executable": true
        }
      ],
      "links": []
    },
    "macos-x86_64-qt5.15": {
      "supersedes": ["macos-x86_64-qt5.6"],
      "requires": {
        "os": "macos",
        "os_version": "10.13",
        "arch": "x86_64"
      },
      "files": [],
      "links": [
        {
          "path": "MultiMC.app/Contents/Frameworks/QtCore.framework/Versions/Current",
          "target": "5"
        }
      ]
    }
  }
}
```

(The lists of the second variant are shortened here; a real `files` list is never empty.)

- `build`: the build number, the same as in the file name.
- `commit`: the Launcher commit the build was made from, lowercase, 40 hex digits.
- `built`: when the build was made.
- `variants`: one or more entries, keyed by variant ID. A variant missing from a build is
  simply not there.

### Variants

A variant is one kind of installable Launcher: a combination of operating system, CPU
architecture and anything else that makes builds differ, such as the Qt version. One
machine may be able to run several variants (a new Mac runs both a Qt 5.6 and a Qt 5.15
build), so the variant is a name of its own, not derived from the machine.

- **IDs** look like `<os>-<arch>-<rest>`, for example `linux-x86_64-qt5`,
  `windows-x86-qt5.6`, `macos-x86_64-qt5.15`. `<os>` and `<arch>` are values from
  `requires`, and must match it; `<rest>` is lowercase letters, digits and dots, in parts
  separated by `-`. At most 64 characters.
- Each Launcher build has its own variant ID compiled in. The Launcher matches IDs only as
  whole strings; it never takes them apart.
- **`supersedes`**: variant IDs this variant replaces. It may name variants that are no
  longer in the build (that is the usual case once an old variant is dropped). In one
  manifest, a variant ID appears in at most one `supersedes` list, a variant doesn't
  supersede itself, and there are no cycles. So from any variant there is a single chain of
  successors.

Which variant a Launcher installs from a build: it follows the chain of successors from its
own variant, and takes the furthest one whose requirements the machine meets. If none of
them qualifies, it takes its own variant if the build has it and the machine meets its
requirements. Otherwise the build is not for this machine, and the Launcher stays where it
is and tells the user which requirement isn't met. This replaces the moves hard-coded in
`getIdealPlatform()`: moving users from one variant to another is a matter of the manifest,
not of a Launcher release.

### Requirements

`requires` says what a machine needs to run the variant. The fields depend on `os`:

| `os` | Fields |
|---|---|
| `windows` | `os`, `arch`, `os_version` |
| `macos` | `os`, `arch`, `os_version` |
| `linux` | `os`, `arch`, `glibc`, `glibcxx`, `qt` |

- `os`: `windows`, `macos` or `linux`.
- `arch`: the CPU architecture of the build: `x86`, `x86_64` or `arm64`. A machine meets it
  when it is the machine's own architecture, and in these cases besides: `x86` on `x86_64`
  Windows, `x86_64` on `arm64` macOS (Rosetta).
- `os_version`: the minimum operating system version. On Windows it is the NT version the
  binaries declare (`5.1` is Windows XP, `6.1` is Windows 7); on macOS the macOS version
  (`10.13`).
- `glibc`: the minimum glibc version, from the highest `GLIBC_` symbol version the binaries
  need.
- `glibcxx`: the minimum libstdc++ version, from the highest `GLIBCXX_` symbol version (for
  example `3.4.21` for `GLIBCXX_3.4.21`).
- `qt`: the minimum Qt version of the system's Qt. The major version must be the same; the
  rest is a minimum (`5.6` accepts 5.6 and later 5.x, never Qt 6).

The values come from what the compatibility gate measures in the binaries, never written by
hand.

### Files

`files` lists every regular file of the variant's installation, sorted by `path` (byte
order).

- `path`: relative to the installation root, the same root the current updater's file lists
  use for that platform. Parts are separated by `/`. No empty parts, no `.` or `..`, no
  leading `/`. Printable ASCII only, without `\ : * ? " < > |`, so the paths work on Windows
  too. At most 512 characters.
- No two paths are the same when compared case-insensitively (macOS and Windows file systems
  usually aren't case-sensitive), and no path is a directory of another file's path.
- `sha256` and `size`: of the file's content. The content file is `files/<first two
  digits>/<sha256>`. Empty files are allowed.
- `executable`: `true` for files installed with mode 0755, `false` for 0644. On Windows it
  is ignored. Nothing else about permissions can be expressed.

### Symbolic links

`links` lists the variant's symbolic links, sorted by `path` (byte order); it is empty when
there are none. macOS frameworks need them (`Versions/Current` and the links next to it).

- `path`: the link itself, by the same rules as a file's path. A path is either a file or a
  link, never both, and the case-insensitive uniqueness rule covers files and links
  together.
- `target`: what the link points to, exactly as the link stores it. It is relative, by the
  same rules as a path except that `..` parts are allowed. Resolved from the link's
  directory, it must stay inside the installation root and name a listed file, a directory
  that holds listed files, or another listed link. Following links never loops.
- Files and links are listed at their real location: no file or link path goes through a
  link (with `Versions/Current` a link to `5`, the files are listed under `Versions/5`).
- A Windows variant has no links.

Links have no content file and no permissions of their own. Empty directories are not part
of format 1; the publisher refuses an installation tree with any other kind of special file
(devices, pipes, sockets).

## The mirror list

```json
{
  "type": "mirrors",
  "format": 1,
  "issued": "2026-11-01T12:00:00Z",
  "mirrors": [
    [
      "https://files.multimc.org/updater/"
    ],
    [
      "https://<bucket host>/updater/",
      "https://<another mirror>/updater/"
    ]
  ]
}
```

- `mirrors`: groups of base URLs, in order of priority. Each group is a list of one or more
  URLs; there is at least one group, and at most sixteen URLs in all, none of them twice.
  Each URL is `https://`, ends with `/`, and has no user info, query or fragment. Each is the
  root of a full copy of the tree.
- `issued`: when the list was written. The Launcher keeps the verified list with the latest
  `issued`, and ignores an older one, so an old list can't be replayed onto it.

It is signed by an online key, so hopper can change it without the offline ceremony. A
mirror is not trusted with anything: every document it serves is checked against its
signature, and every content file against its hash.

How the Launcher picks a mirror:

- It tries the groups in order: a group is only used when every mirror in the groups before
  it has failed.
- Within a group it picks the mirrors in random order, so the load spreads across them.
- It stays with the mirror that answers for the whole update check and download, and moves
  on to the next one only when that mirror fails (unreachable, an error, a document or file
  that doesn't verify). Fetching everything from one mirror avoids mixing documents from
  mirrors that are at different points of a sync.
- After the list, it tries the compiled-in hosts that aren't in it, in their compiled-in
  order.

## The validator

A script in hopper, used by the publisher before anything is written and by the tests. It
checks one document (with its signature, given the keys to trust) or a whole tree (given the
embedded floor keys document a Launcher would have), against everything in this document,
and exits non-zero on the first problem. It uses Python's standard library only; signatures
are checked with the `openssl` CLI.
