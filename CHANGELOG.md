# Changelog

All notable changes to rss-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.3 — 2026-09-25

The package builds with novo 0.11.  Every body is still `todo()`.

- The lock file moves url-nv 0.1.1 to 0.1.3.  url-nv 0.1.1 writes into
  lists through names that are not declared `var`, which novo 0.11
  refuses (E2038), so this package did not build with novo 0.11 against
  it.  No requirement in the manifest changed.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md).

`RssFault` now declares the `impl Error` its own `Result` positions
require.  `Result<T, E>` has carried the bound `E: Error` since SPEC
§ 3.4, and the compiler enforced it only when `E` was declared in the
module that named it — so `Result<_, rsserror.RssFault>` was accepted
across modules with no impl anywhere.  The impl is the signature this
package always meant; nothing else about the interface changed.  The
xml-nv range moves to `^0.0.2`, the version whose `XmlFault` carries
its own impl.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `rssfeed` — one feed value for all three formats: `RssFeed`,
  `RssEntry`, `RssText` with its kind, `RssLink` with its `rel`,
  `RssPerson`, `RssCategory`, `RssEnclosure`, `RssStamp`; a
  constructor for every one of them and a `with_`/`push_` builder for
  the two large ones; and the questions a feed reader asks —
  `entry_url`, `entry_body`, `entry_when`, `newest`,
  `sorted_entries`, `self_url`, `site_url`.
- `rssread` — `read`, `read_with`, `read_xml`, the two format-pinned
  readers, and the streaming door: `stream`, `next_entry`, `head_of`.
  `RssTolerance` is the policy and `RssOptions` is the budget.
- `rssjson` — JSON Feed 1.0 and 1.1 over `JsonValueH`, the standard
  library's own JSON value, read and written; `unread_members` names
  what does not survive a round trip.
- `rsswrite` — `write` for any format into a caller's buffer, the three
  format-specific writers, the head/entry/tail fragment writers, and
  `writable`, which answers every field the target format cannot carry
  before a caller commits to it.
- `rssdate` — RFC 822 parsed and printed, RFC 3339 through
  calendar-nv, and `parse_any` over `formats()`, which is the ordered
  contract rather than a private list.
- `rsserror` — `RssNote` and `RssFault` as one vocabulary, and
  `stops_strict`, the single table that decides which notes a strict
  read refuses.

### Known

- **Every human-readable string is an `RssText`**, never a `Str`,
  because three formats spell "is this markup" three ways and a
  flattened string cannot be rendered safely in either direction.
- **The tolerance is an enum and the repairs are a list.** feedparser's
  single `bozo` flag is recomputed from the list by
  `rssread.would_fail_strict` for anyone who wants it.
- **RSS 1.0 / RDF and Atom 0.3 are refused by name**, because reading
  either as its nearest neighbour produces a wrong answer that does
  not look like a failure.
- **Sanitising, transcoding, HTML entity expansion and the thirty
  extension namespaces are out**, and the README says why for each.
  Four extensions are in: `<content:encoded>`, `dc:date`,
  `dc:creator` and `<atom:link rel="self">`.
- **Nothing fetches.** Conditional GET, redirects and ETag storage
  belong to a `host` package over `std.http`.
- **An RSS 2.0 write emits `<atom:link rel="self">`**, because RSS 2.0
  has no element for a feed's own address and a feed that cannot state
  it cannot be re-found from its own contents.
- **JSON Feed is not streamable**; `rssread.stream` says so rather
  than pretending.
- **No device claim**: a feed is lists of strings, and the JSON half
  rides on a standard library value that is not available at
  `@tier(embedded)`.
- **Three `core` dependencies** — xml-nv, calendar-nv, url-nv — and
  html-nv deliberately absent.
