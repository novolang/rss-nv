# rss-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

Feeds, in the three formats anybody publishes: RSS 2.0, Atom 1.0 and
JSON Feed 1.1.  All three read into **one** value and all three are
written back out of it, so a program that reads feeds never learns
which one it got, and a program that publishes one picks its format at
the last moment.

It is what you reach for when you are writing a feed reader, a
planet-style aggregator, a podcast client, or a site generator that
has to emit a feed people can subscribe to.  Nothing here fetches
anything: you hand it bytes and it hands you a value.

Six modules, and a reader should know which one they are on.

| surface | module | reach for it when |
| --- | --- | --- |
| the **feed** | `rssfeed` | you are building a feed, or asking one a question |
| the **reader** | `rssread` | you have bytes and want a feed |
| the **JSON half** | `rssjson` | the bytes were `application/feed+json` |
| the **writer** | `rsswrite` | you are publishing a feed |
| the **dates** | `rssdate` | a feed's date string did not parse |
| the **notes** | `rsserror` | you are reporting what was odd about somebody's feed |

## Adding it, and checking it

```bash
novo pkg add rss-nv            # into your novo.toml
novo pkg build                 # type- and effect-check the package
novo test --isolate tests/rssread_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: rss-nv.<module>.<fn>`.  They
turn green one at a time as bodies land.

## The one example that will work

```novo
use rssfeed
use rssread

fn main() [io]
    match rssread.read(bytes_from_somewhere())
        Err(f) => println("not a feed")
        Ok(r)  =>
            println(r.feed.title.text)
            for e in rssfeed.sorted_entries(r.feed)
                println("  ${e.title.text} — ${rssfeed.entry_url(e)}")
            for n in r.notes
                println("  note: ${n.detail}")
```

Nothing in that changes if the bytes were RSS, Atom or JSON Feed.

## The load-bearing interface

`rssfeed.RssText` — a string, and what kind of string it is.

```novo norun:pseudo
pub struct RssText
    text: Str
    kind: RssTextType   // RssTextPlain | RssTextHtml | RssTextXhtml
    base: Str
    lang: Str
```

Every human-readable string in this package is one of these and none
of them is a `Str`.  That looks like ceremony until you try to render
a feed.

RSS 2.0's `<description>` is escaped HTML by universal convention and
by no specification — the 2.0 document does not say, and the two
readings differ for every entry that contains a `<`.  Atom does say,
in an attribute, and says it three ways: `type="text"` is characters,
`type="html"` is escaped markup, `type="xhtml"` is *live* markup
inside a wrapper `<div>` that is not itself content.  JSON Feed splits
the question into two members, `content_text` and `content_html`, and
permits an item to carry either or both.

One question, three spellings.  A reader that flattened them to `Str`
hands its caller a string that cannot be displayed safely in either
direction: escape it and the end user sees `<p>` in their reading
list; do not escape it and you have stored cross-site scripting from
whatever the feed said.  So the answer travels with the text, and
`rssfeed.text_is_markup` is the one question a renderer has to ask.

The second decision the rest follows from is that **the tolerance is
an enum and the repairs are a list**.  feedparser has a single `bozo`
flag and a single `bozo_exception`; a feed with nine small deviations
and a feed with one fatal one both come back with `bozo = 1`, so a
consumer cannot tell "I repaired nine things" from "I gave up".  Here
a read answers a `Result` *and* a list of `rsserror.RssNote`, and
`rsserror.stops_strict` is one total function from a note kind to
whether `RssStrict` turns it into a fault.  The policy is a table, not
a branch in each of three readers, and `rssread.would_fail_strict`
recomputes feedparser's flag from the list for anyone who wants it.

## Which of feedparser's normalisations are in

**In**, and each one files a note saying it happened:

- The date parse.  `rssdate.parse_any` tries the shapes in
  `rssdate.formats()`, in order, so an RSS feed carrying an RFC 3339
  date and an Atom feed carrying an RFC 822 one both read
  (`RssNoteDateWrongFormat`).  Two-digit years widen under RFC 2822's
  rule (`RssNoteTwoDigitYear`); an unknown alphabetic zone reads as
  `+0000` (`RssNoteUnknownTimezone`).
- Relative references resolved against `xml:base`, or against the
  address the caller says the document came from
  (`RssNoteRelativeUrl`), with `RssNoteUnresolvedUrl` when there was
  no base to resolve against.
- `rel` filled in on an Atom link that has none — RFC 4287 § 4.2.7.2
  says it is `alternate`.
- An RSS item with no `<guid>` taking its id from its `<link>`
  (`RssNoteSyntheticId`).
- `address (Name)` split into an address and a name
  (`RssNoteAuthorSplit`).
- An RSS `<description>` read as HTML (`RssNoteContentTypeGuessed`).
- The named extensions that make a real RSS feed usable:
  `<content:encoded>`, `dc:date`, `dc:creator`, and `<atom:link
  rel="self">` inside an RSS channel.  `RssOptions.read_extensions`
  turns them off.
- Elements in the wrong namespace read anyway
  (`RssNoteMismatchedNamespace`).

**Out**, on purpose:

- **Sanitising.**  feedparser strips scripts and event handlers out of
  content it returns.  This package carries the text and says what
  kind it is; sanitising markup is html-nv's job and it should be the
  consumer's explicit decision, made with a visible allow-list, rather
  than a thing a feed parser did on the way past.
- **Character-set detection and transcoding.**  feedparser sniffs the
  encoding from the HTTP header, the XML declaration and the bytes,
  and transcodes.  Here the bytes are read as UTF-8, the declared
  encoding is reported in `RssRead.declared_encoding`, and a document
  that declared something else files `RssNoteEncodingIgnored`.  A
  caller that needs to re-decode has what it needs to do so; a `core`
  package that carried a charset table would be carrying a much larger
  table than a feed reader needs.
- **HTML entity expansion.**  XML's five predefined entities are
  expanded; HTML's 2 231 named references are not
  (`RssNoteUndeclaredEntity`).  That table is html-nv's.
- **Microformat and namespace scraping.**  feedparser reads about
  thirty extension namespaces — iTunes, Media RSS, GeoRSS, CreativeCommons,
  and more.  Four are read here (above); the rest are not, because a
  podcast client wanting the iTunes vocabulary is better served by a
  package that models it properly than by fields bolted onto a generic
  entry.
- **Anything that fetches.**  No conditional GET, no redirect
  following, no ETag store.  Those are a `host` package's, over
  `std.http`, and this one has no effects with which to do them.
- **RSS 1.0 / RDF and Atom 0.3**, which are refused by name.  See
  below.

## What is refused rather than approximated

- **RSS 1.0 / RDF** (`RssRdfRefused`).  It is an RDF vocabulary, not a
  dialect of RSS 2.0: its `<item>`s are siblings of the `<channel>`
  rather than children, so reading it as RSS 2.0 produces a feed with
  a title and no entries and nothing that looks like a failure.
- **Atom 0.3** (`RssAtomDraftRefused`).  0.3 has `<issued>`,
  `<modified>` and `<created>`, which are three dates where 1.0 has
  two, and every mapping between them is wrong for some feeds.
- **A JSON Feed with no `version`** (`RssNotJsonFeed`), because
  without it the value is some other JSON object entirely.

## What each format cannot carry

`rsswrite.writable` answers this for a given feed before a write, one
fault per field, so a site generator learns at startup rather than at
the end of a build.

To **RSS 2.0**: XHTML content, `contributors`, an entry `updated`, a
feed `id` other than its self link, any link `rel` other than
`alternate`, `self` or `enclosure`.

To **Atom 1.0**: nothing this package models.  It is the format
`write` picks when a caller expresses no preference.

To **JSON Feed 1.1**: a category's `scheme`, a link's `hreflang`,
`contributors`, `rights`, `ttl_minutes`, any `rel` other than
`alternate` and `next`.

`RssWriteOptions.lossy` turns each refusal into a dropped field and a
note, for the caller who genuinely wants RSS 2.0 and has read the list.

One deliberate impurity: an RSS 2.0 write emits `<atom:link
rel="self">`.  RSS 2.0 has no element for a feed's own address, a feed
that does not state it cannot be re-found from its own contents, and
the Atom element is what every RSS feed in the wild already uses for
this.

## A 40 MB archive feed

`rssread.stream` reads as far as the first entry — so the feed's title
and `updated` stamp are available immediately — and `rssread.next_entry`
answers one entry at a time.  Nothing accumulates.  On the writing
side `rsswrite.write_head`, `write_entry` and `write_tail` are the
same shape, so a program streaming a feed out of a database never
builds a whole `RssFeed`.

JSON Feed is not streamable this way; the value arrives whole, and
`stream` says so rather than pretending.

## Why `core`

Nothing here reads or writes anything: a reader takes bytes the caller
already has, and a writer appends to a buffer the caller owns.  That
is what makes the tolerance policy testable — every one of the repairs
above is a note on a value, asserted against a string in a test, with
no server in the room.

No device claim.  A feed is a list of entries each holding several
lists of strings; xml-nv's tree is one allocation per document and
makes no device claim either; and the JSON half rides on `JsonValueH`,
which the standard library documents as unavailable at
`@tier(embedded)`.  A microcontroller does not read feeds.

## Dependencies

Three, all `core`.

- **xml-nv** — the scanner and the tree.  `xmlparse.next_event` is the
  streaming door `rssread.stream` is built on, `xmltree` is what the
  whole-document readers walk, and `xmlwrite`'s three escape rules —
  character data, attribute values, comments — are what a feed writer
  that rolled its own would get wrong.
- **calendar-nv** — the civil date-time an `RssStamp` holds, and RFC
  3339.  RFC 822, which RSS 2.0 specifies and calendar-nv has no
  parser or printer for, lives in `rssdate`, along with the tolerant
  multi-format parse — which does not belong in a date library,
  because a date library whose one entry point accepts four different
  shapes cannot be used to validate anything.
- **url-nv** — RFC 3986 § 5.3, which is what resolving a relative
  reference against `xml:base` is.

Not **html-nv**.  This package carries text and says what kind it is;
it does not sanitise, escape or render it.  A consumer that displays a
feed needs html-nv and should depend on it directly — putting it here
would make every feed poller that only reads titles and dates pay for
an HTML tokenizer.

## Ports

[rss](https://github.com/rust-syndication/rss) and
[feedparser](https://github.com/kurtmckee/feedparser) are the
reference implementations; feedparser's own test corpus is the oracle
for the tolerance policy, and its `sgml` and `http` suites are out of
scope for the reasons above.
[JSON Feed 1.1](https://www.jsonfeed.org/version/1.1/) is read from
its specification.

## Licence

Apache-2.0.
