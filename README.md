# tralbum

A Python client for Bandcamp's public data, built for research use.

![architecture](docs/architecture.png)

## Scope

Fetches and parses five things:

- a label's or artist's catalogue
- a release page: metadata, tags, credits, and the `mp3-128` stream URLs
- the supporter list on a release page, which is the set of fans who bought it
- a fan's collection, paginated, with the time each item was added
- a fan's wishlist

Everything it returns comes from pages that are public to a logged-out
visitor.

Out of scope:

- Downloading or storing audio. The library returns stream URLs and nothing
  more.
- Authenticated endpoints, purchases, account actions.
- Recommendation or analysis logic. That belongs downstream.

## Timestamps

Every item in a fan collection carries a token :

```
data-token="1770754734:1227574518:p::"
            |          |          |
            unix sec   item_id    type code (p=package, a=album, t=track)
```

The first field is when the item entered the collection. The
`collection_items` endpoint carries the same fact as a `purchased` field
formatted `10 Feb 2026 20:18:54 GMT`, and uses the token as its
`older_than_token` pagination cursor.

Existing clients discard this. `tralbum` preserves both the parsed datetime
and the raw token string, and the test suite asserts that it does. Without
the timestamp there is no way to reconstruct what was knowable about a
release on the day it came out, which rules out most interesting questions
about adoption.

## Relationship to bandcamp-fetch

[bandcamp-fetch](https://github.com/patrickkfkan/bandcamp-fetch) by Patrick
Kan is the most complete existing map of Bandcamp's undocumented endpoints
and is MIT licensed. Field mappings in `tralbum/parse/` are ported from it.

## Install

```
pip install -e ".[dev]"
```

Python 3.11 or later.

## Usage

```python
from tralbum import Client

client = Client(archive="./raw", rate=1.0)   # requests per second

for release in client.catalog("ilianbe.bandcamp.com"):
    print(release.title, release.release_date)

rel = client.release("https://ilianbe.bandcamp.com/album/example")
print(rel.tags, rel.label, [t.stream_url for t in rel.tracks])

for fan in client.supporters(rel.tralbum_id):
    print(fan.username, fan.favourite_track_id)

for item in client.collection("some-fan"):
    print(item.added_at, item.tralbum_id, item.item_type)
```

Every call is a generator that handles its own pagination and writes the raw
response to the archive before parsing it.

## Architecture

### Raw responses are archived before parsing

Every HTTP response is gzipped to disk under the archive directory, keyed by
URL and fetch time, before any parsing happens.

### The library holds no crawl state

`tralbum` fetches and parses. Crawl frontiers, databases, resume logic, and
scheduling belong in the consuming project. The split keeps the library
testable without a network and useful to people who are not running the same
pipeline.

### Parsers do no I/O

Functions in `parse/` take bytes and return dataclasses. No requests, no file
access, no clock.

### One rate limiter per process

A single token bucket shared by every endpoint, default one request per
second. There is no flag to disable it. A crawl that is too slow should be
narrowed, not accelerated.

## Layout

```
tralbum/
  client.py          HTTP, TLS impersonation, rate limiting, retries
  archive.py         raw response store
  endpoints/         one module per endpoint, each a paginating generator
  parse/             pure functions: bytes in, dataclass out
  models.py          Release, Track, Fan, CollectionItem
  cli.py             tralbum fetch <url>
tests/
  fixtures/          saved pages
docs/
  endpoints.md       reverse-engineering notes with example payloads
```

## Vocabulary

Used consistently in code, docs, and commit messages:

- page: a Bandcamp subdomain account, flagged as artist or label
- release: a tralbum with `item_type = "a"`
- track: a tralbum with `item_type = "t"`
- item: what a fan buys, a release or a track in a given package
- fan: an account with a public collection
- supporter: a fan listed on a release page as having bought it
- label: the hosting page when it is flagged as a label, otherwise the string
  `self-released`, which is a value and not a missing field

## Contributing

Tests never touch the network. `tests/fixtures/` holds saved HTML and JSON
captured from real pages, and every parser test reads a fixture. A pull
request that adds a parser adds a fixture with it.

Fixture filenames record what the page was and when it was captured, for
example `release_va-compilation_2026-11-18.html`. Capture only what the page
shows publicly.

Changes to `models.py` affect everyone downstream, so open an issue before
changing a field rather than sending the rename directly.

`docs/endpoints.md` is written as endpoints are reverse engineered, not
afterwards. An endpoint without notes is unfinished.

## Politeness and legal position

Scraping is against Bandcamp's terms of service. This library operates at
research scale and does not pretend otherwise.

- One request per second, no parallel crawling of the same host.
- Aggressive caching. The same URL is never fetched twice in a run.
- A `User-Agent` identifying the project with a contact address.
- No redistribution of audio, raw collection data, or fan identifiers.
  Anything published alongside a paper is hashed identifiers and derived
  features.
- Bandcamp is a small company that pays artists well. If they ask us to stop,
  we stop.

## License

MIT.
