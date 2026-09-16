# rss-nv

A **feed** is a file a site publishes listing its recent items, so that
a program can follow the site without a person visiting it. Three
formats are in use:
[RSS 2.0](https://www.rssboard.org/rss-specification),
[Atom 1.0](https://www.rfc-editor.org/rfc/rfc4287) and
[JSON Feed 1.1](https://www.jsonfeed.org/version/1.1/). This package
reads all three into one value and writes that value back out as any of
them, so a program that reads feeds never learns which format it got,
and a program that publishes one chooses the format at the last moment.
It is a port of the Rust crate
[rss](https://github.com/rust-syndication/rss) and of Python's
[feedparser](https://github.com/kurtmckee/feedparser).

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What a feed is

A feed has a title, an address of its own, a last-updated stamp and a
list of **entries**. An entry has an identifier, a title, a link, one
or two dates, and a body. RSS calls an entry an item and Atom calls it
an entry; this package calls it an entry.

Feeds in the wild are frequently invalid, and a reader that refused
them would read almost nothing. Reading one is therefore tolerant: this
package repairs what it can and records each repair as a **note**, a
named kind with a position and the text it was about.

The three formats disagree about what a human-readable string is. RSS
2.0's `<description>` is escaped HTML by universal convention and by no
specification, so the document does not say and the two readings differ
for every entry containing a `<`. Atom says, in an attribute, and says
it three ways: `type="text"` is characters, `type="html"` is escaped
markup, and `type="xhtml"` is live markup inside a wrapping `<div>`
that is not itself content (RFC 4287 section 3.1). JSON Feed splits the
question into `content_text` and `content_html`, and an item may carry
either or both.

So every human-readable string in this package is an `RssText`: the
text, and what kind of text it is. A reader that flattened the three
spellings into a plain string would hand its caller something that
cannot be displayed correctly either way: escape it and a person sees
`<p>` in their reading list, do not escape it and the feed has injected
markup into the page.

A **stamp** is a date and a time with an offset. RSS 2.0 specifies RFC
822 dates and Atom specifies RFC 3339 ones, and feeds carry each in the
other's place often enough that this package tries both.

Every function in this package performs no input and no output. Nothing
here fetches a feed: the caller hands it bytes.

## Install

```
novo pkg add rss-nv
```

## Example

```novo
use rsserror
use rssfeed
use rssread

fn main() [io]
    // Bytes as they arrived, in any of the three formats.
    let source = "<rss version=\"2.0\"><channel><title>Example</title></channel></rss>"

    match rssread.read(source)
        Err(f) => println("not a feed: ${rsserror.message(f)}")
        Ok(r)  =>
            // The same fields whichever format the bytes were.
            println(r.feed.title.text)

            // Entries newest first, with the link a reader opens.
            for e in rssfeed.sorted_entries(r.feed)
                println("  ${e.title.text} — ${rssfeed.entry_url(e)}")

            // Everything the read repaired, one note each.
            for n in r.notes
                println("  note: ${rsserror.note_text(n)}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: rss-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `rsserror` | The notes a read files, the faults that stop one, and which notes strict reading turns into faults. |
| `rssfeed` | The feed and the entry as values, the builders over them, and the questions a consumer asks. |
| `rssdate` | RFC 822 and RFC 3339 dates, the tolerant parse over both, and the printers. |
| `rssread` | Reading: format detection, the whole-document readers, and the streaming reader. |
| `rssjson` | The JSON Feed half, over the standard library's JSON value. |
| `rsswrite` | Writing: a feed into a caller's buffer as any of the three formats, and what each format cannot carry. |

## How to choose an entry point

**`rssread.read` takes bytes and answers a feed.** It detects the
format. `read_with` takes options. `read_rss20` and `read_atom10` take
a format the caller already knows, and `read_xml` takes a document
xml-nv has already parsed.

**`rssread.stream` reads as far as the first entry**, so a feed's title
and stamp are available immediately, and `next_entry` answers one entry
at a time with nothing accumulating. Use it for a large archive feed.
JSON Feed is not streamable this way and `stream` says so.

**`rsswrite.write` writes a whole feed.** `write_head`, `write_entry`
and `write_tail` are the streaming form, for a program producing a feed
out of a database without building the whole value.

**`rsswrite.writable` asks what a format cannot carry**, before a
write, so a site generator learns at start-up rather than at the end of
a build.

**`rssfeed.text_is_markup` is the one question a renderer must ask** of
every string it displays.

## The rules a user needs

1. **Every human-readable string carries its kind.** `RssTextPlain` is
   characters, `RssTextHtml` is escaped markup, `RssTextXhtml` is live
   markup. Ask `rssfeed.text_is_markup` before displaying one.
2. **This package does not sanitise.** It carries the text and says
   what kind it is. Removing scripts and event handlers from markup is
   [html-nv](https://novo-lang.org/packages/html-nv)'s work and should
   be a consumer's explicit decision with a visible allow-list.
3. **A read answers a result and a list of notes.** The result says
   whether the document could be read at all. The notes say what was
   repaired. A single flag cannot distinguish nine small repairs from
   one fatal failure.
4. **`rsserror.stops_strict` is the whole policy.** It is one total
   function from a note kind to whether `RssStrict` turns that note
   into a fault. `rssread.would_fail_strict` recomputes feedparser's
   single flag from the note list for a caller that wants it.
5. **Dates are parsed by shape, not by which element they were in.**
   `rssdate.parse_any` tries `rssdate.formats()` in order, so an RSS
   feed carrying an RFC 3339 date reads, and files
   `RssNoteDateWrongFormat`.
6. **A two-digit year is widened under RFC 2822's rule**, and files
   `RssNoteTwoDigitYear`. An alphabetic zone outside `UT`, `GMT` and
   the four North American pairs reads as `+0000`, which is what RFC
   822 section 5.2 says, and files `RssNoteUnknownTimezone`. Both are
   somebody's guess and both are recorded.
7. **A relative reference is resolved against `xml:base`, or against
   `RssOptions.base`.** Set `base` to the address the document was
   fetched from. Without one, a relative reference is kept as written
   and files `RssNoteUnresolvedUrl`. Turn `resolve_relative` off when
   round-tripping a feed unchanged.
8. **An RSS item with no `<guid>` takes its identifier from its
   `<link>`**, and files `RssNoteSyntheticId`. Two entries with one
   identifier are both kept, in arrival order, and file
   `RssNoteDuplicateId`: deciding which wins is the consumer's.
9. **An Atom link with no `rel` is `alternate`** (RFC 4287 section
   4.2.7.2).
10. **Four extensions are read, and `read_extensions` turns them off.**
    `<content:encoded>`, Dublin Core's `dc:date` and `dc:creator`, and
    an `<atom:link rel="self">` inside an RSS channel. They are what
    makes a real RSS feed usable.
11. **The bytes are read as UTF-8 and nothing is transcoded.** A
    document declaring another encoding files
    `RssNoteEncodingIgnored`, and `RssRead.declared_encoding` carries
    what it declared, so a caller can decode again itself.
12. **XML's five predefined entities are expanded and HTML's are not.**
    An undeclared entity is left as written and files
    `RssNoteUndeclaredEntity`. The 2231 named references are html-nv's
    table.
13. **Three inputs are refused rather than approximated.**

    | Refused | Why |
    | --- | --- |
    | RSS 1.0, which is RDF | Its items are siblings of the channel rather than children, so reading it as RSS 2.0 gives a feed with a title, no entries and nothing that looks like a failure |
    | Atom 0.3 | It has `issued`, `modified` and `created` where 1.0 has two dates, and every mapping between them is wrong for some feeds |
    | A JSON Feed with no `version` | Without it the value is some other JSON object |

14. **Set `max_entries` when only the newest matter.** A reader showing
    ten posts never allocates the other nine thousand, and the read
    files `RssNoteEntriesTruncated`.
15. **An RSS 2.0 write emits `<atom:link rel="self">`.** RSS 2.0 has no
    element for a feed's own address, so a feed that does not state it
    cannot be re-found from its own contents, and this is what every
    RSS feed in the wild already uses.

## What each format cannot carry

`rsswrite.writable` answers this for a given feed and target, one fault
per field. `RssWriteOptions.lossy` turns each refusal into a dropped
field and a note, for a caller that wants RSS 2.0 and has read the
list.

| Target | Cannot carry |
| --- | --- |
| RSS 2.0 | XHTML content, contributors, an entry's `updated`, a feed identifier other than its self link, and any link relation other than `alternate`, `self` or `enclosure` |
| Atom 1.0 | nothing this package models. It is what `write` picks when the caller expresses no preference |
| JSON Feed 1.1 | a category's scheme, a link's language, contributors, rights, a time-to-live, and any link relation other than `alternate` or `next` |

## Limits

| `RssOptions` field | `default_options()` |
| --- | --- |
| `tolerance` | `RssLenient` |
| `base` | empty |
| `max_entries` | 0, meaning no limit |
| `max_bytes` | 0, meaning no limit |
| `max_depth` | 64 |
| `resolve_relative` | true |
| `read_extensions` | true |

A feed is three levels deep, so a document needing more than 64 is not
a feed.

## What is not included

- **Anything that fetches.** No conditional request, no redirect
  following, no ETag store. This package declares no effects.
- **Sanitising.** See rule 2.
- **Character set detection and transcoding.** See rule 11. A table of
  character sets is much larger than a feed reader needs.
- **HTML entity expansion.** See rule 12.
- **The thirty extension namespaces feedparser reads**, such as iTunes,
  Media RSS and GeoRSS. Four are read, listed in rule 10. A podcast
  client wanting the iTunes vocabulary is better served by a package
  that models it than by fields bolted onto a generic entry.
- **RSS 1.0 and Atom 0.3.** See rule 13.
- **A build for a microcontroller.** A feed is a list of entries each
  holding several lists of strings, and the JSON half rides on a host
  handle, so this package does not build for a microcontroller with no
  heap allocator.

## Related packages

- [xml-nv](https://novo-lang.org/packages/xml-nv) supplies the scanner
  the streaming reader is built on, the tree the whole-document readers
  walk, and the three escape rules a feed writer needs. This package
  depends on it.
- [calendar-nv](https://novo-lang.org/packages/calendar-nv) is the
  civil date and time a stamp holds, and RFC 3339. RFC 822, which RSS
  2.0 specifies, lives here in `rssdate`, along with the tolerant
  parse: a date library whose one entry point accepts four shapes
  cannot be used to validate anything. This package depends on it.
- [url-nv](https://novo-lang.org/packages/url-nv) resolves a relative
  reference against a base, which is RFC 3986 section 5.3. This
  package depends on it.
- [html-nv](https://novo-lang.org/packages/html-nv) is what a consumer
  displaying a feed needs, for escaping, sanitising and the named
  character references. This package does not depend on it, so a feed
  poller reading only titles and dates does not link an HTML tokenizer.
- [mime-nv](https://novo-lang.org/packages/mime-nv) is what decides
  whether a fetched body is a feed before it is parsed.
- `std.json` in the standard library is the value the JSON Feed half
  reads and writes.

## Tests

```bash
novo test tests/rsserror_tests.nv   # the notes, and which are strict
novo test tests/rssfeed_tests.nv    # the feed value and its questions
novo test tests/rssdate_tests.nv    # RFC 822, RFC 3339, and the tolerant parse
novo test tests/rssread_tests.nv    # the three readers and the repairs
novo test tests/rssjson_tests.nv    # JSON Feed
novo test tests/rsswrite_tests.nv   # the writers and what each format refuses
```

The normative sources are the RSS 2.0 specification, RFC 4287 for Atom
1.0, and the JSON Feed 1.1 specification. The reference implementations
are the Rust crate `rss` and Python's feedparser, and feedparser's own
test corpus is the oracle for the tolerance policy. Its character-set
and HTTP suites are out of scope for the reasons above.

The suite asserts that the same feed read from all three formats gives
the same value, that each repair in rule 5 through rule 12 files the
named note, that RSS 1.0 and Atom 0.3 are refused by name, that a
strict read stops at the first note `stops_strict` answers true for,
and that `writable` names every field a target cannot carry.

The tests compile today and fail at run, each on the
`not implemented: rss-nv.<module>.<fn>` panic that is its body. That is
the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| Every `pub struct` and `pub enum` in the six modules | the types are declared |
| `rsserror.fault`, `.note`, `.stops_strict`, `.strict_kinds` | no |
| `rsserror.note_name`, `.note_text`, `.kind_name`, `.message`, `.is_recoverable` | no |
| `rsserror.notes_for_entry`, `.notes_of_kind` | no |
| `rssfeed.empty_feed`, `.empty_entry`, `.plain`, `.html`, `.xhtml`, `.text_in` | no |
| `rssfeed.link`, `.link_full`, `.person`, `.category`, `.enclosure`, `.stamp` | no |
| `rssfeed.with_id`, `.with_title`, `.with_subtitle`, `.with_updated`, `.with_language`, `.with_generator` | no |
| `rssfeed.push_link`, `.push_author`, `.push_category`, `.push_entry` | no |
| `rssfeed.entry_with_id`, `.entry_with_title`, `.entry_with_content`, `.entry_with_summary`, `.entry_with_url` | no |
| `rssfeed.entry_with_published`, `.entry_with_updated` | no |
| `rssfeed.entry_push_link`, `.entry_push_author`, `.entry_push_category`, `.entry_push_enclosure` | no |
| `rssfeed.text_is_markup`, `.link_with_rel`, `.links_with_rel` | no |
| `rssfeed.self_url`, `.site_url`, `.entry_url`, `.entry_body`, `.entry_when` | no |
| `rssfeed.newest`, `.compare_stamps`, `.sorted_entries`, `.entry_by_id` | no |
| `rssfeed.format_name`, `.format_mime` | no |
| `rssdate.formats`, `.parse_rfc822`, `.parse_rfc3339`, `.parse_any`, `.shape_of` | no |
| `rssdate.format_rfc822`, `.format_rfc3339`, `.zone_offset`, `.widen_year`, `.epoch_seconds` | no |
| `rssread.default_options`, `.with_tolerance`, `.with_base`, `.with_max_entries` | no |
| `rssread.sniff`, `.read`, `.read_with`, `.read_xml`, `.read_rss20`, `.read_atom10` | no |
| `rssread.stream`, `.next_entry`, `.head_of`, `.stream_notes` | no |
| `rssread.notes_of`, `.would_fail_strict` | no |
| `rssjson.versions`, `.read_value`, `.read_text`, `.is_feed`, `.unread_members` | no |
| `rssjson.to_value`, `.entry_to_value`, `.entry_from_value` | no |
| `rsswrite.default_options`, `.compact`, `.with_lossy`, `.with_self_url`, `.with_max_entries` | no |
| `rsswrite.writable`, `.write`, `.write_rss20`, `.write_atom10`, `.write_json11` | no |
| `rsswrite.write_entry`, `.write_head`, `.write_tail`, `.write_len` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
