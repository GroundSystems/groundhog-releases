---
title: Python SDK
description: Install and use the M1 groundhog-sdk Python client for ingest, replay, query, and catalog access.
---

The `groundhog-sdk` package is the synchronous Python client for the Groundhog v1 public API. It
connects to a running `groundhog` process over a Unix domain socket. Each SDK operation maps to
one public HTTP operation; durability, ordering, idempotency, and query confinement remain
Groundhog responsibilities.

## Requirements

- Python 3.10 or newer.
- A running M1-compatible `groundhog serve` process.
- Access to its Unix socket and, when configured, its bearer token.

The M1 SDK does not install, start, stop, or supervise Groundhog. Initialize and serve the
deployment separately:

```sh
groundhog init ./demo
groundhog serve --config ./demo/groundhog.toml
```

## Installation

Install the published distribution from PyPI:

```sh
python -m pip install groundhog-sdk
```

The distribution is named `groundhog-sdk`; Python code imports `groundhog_sdk`.

For local development, install from a checkout of the SDK repository with
`python -m pip install .`.

## Connect

```python
from groundhog_sdk import Ground

ground = Ground("unix:demo/data/ground.sock")
```

`Ground` accepts:

```python
Ground(
    endpoint=None,
    token=None,
    *,
    timeout=30.0,
    max_retries=3,
)
```

`endpoint` must use the `unix:` scheme. `unix:/absolute/path/ground.sock` and
`unix:relative/path/ground.sock` are both accepted. With no explicit endpoint, the client uses
`GROUND_URL`, then `unix:data/ground.sock`. With no explicit token, it uses `GROUND_TOKEN`.
Explicit arguments take precedence over environment variables.

```sh
export GROUND_URL="unix:demo/data/ground.sock"
export GROUND_TOKEN="replace-with-a-secret"
```

```python
ground = Ground()
```

`timeout` is the transport timeout in seconds. `max_retries` is the number of attempts after the
initial request when a connection fails or Groundhog refuses admission with HTTP 429. Ingest
also retries a temporarily unavailable writer with capped exponential backoff. A 429 response's
`Retry-After` value is honored.

## Build events

The constructors produce exactly the connector-owned event fields. Payload values are retained
without reshaping.

```python
from groundhog_sdk import deleted, native, upserted

created = upserted(
    "customers",
    "cus_123",
    {"id": "cus_123", "name": "Ada"},
    occurred_at="2026-07-14T16:30:00Z",
)

removed = deleted(
    "customers",
    "cus_456",
    {"id": "cus_456"},
)

paid = native(
    "invoices",
    "in_789",
    "invoice.paid",
    {"id": "in_789", "amount": 4200},
)
```

The SDK validates stream names, non-empty record keys and kinds, UTF-8 byte limits, UTC
`occurred_at` syntax, calendar dates, JSON-serializable payload values, finite numbers, malformed
Unicode, and the payload nesting limit before sending. Groundhog performs the authoritative
validation.

## Ingest an atomic batch

```python
receipt = ground.send(
    "stripe",
    [created],
    batch_id="stripe/customers/page-1",
)

print(receipt.status)          # "committed" or "duplicate"
print(receipt.batch_digest)
print(receipt.events)
print(receipt.first_event_id)
print(receipt.last_event_id)
```

`send(source, events, batch_id)` maps to one JSON `POST /v1/events` request. A batch must contain
between 1 and 10,000 events and its encoded body must not exceed 32 MiB. `source` follows the
Groundhog source-name grammar; `system` is reserved. `batch_id` must be non-empty, at most 256
UTF-8 bytes, and must not begin with the reserved `groundhog/` prefix.

The entire batch is buffered and encoded once before the first attempt. If the connection closes
before a response arrives, the SDK retries those identical bytes with the same `(source,
batch_id)`. The result converges to one durable batch:

- `committed` means this request durably appended the batch;
- `duplicate` means identical content was already durable and is equally safe;
- `IdempotencyConflict` means the batch ID was already committed with different content and is
  never retried.

Advance a source cursor only after receiving a `BatchReceipt` with either successful status.

## Replay events

```python
page = ground.events(
    after=None,
    source="stripe",
    stream="customers",
    kind="upserted",
    limit=1000,
)

for event in page.events:
    print(event["event_id"], event["record_key"], event["payload"])

cursor = page.next_after
```

`events()` maps to one `GET /v1/events` request. Returned events are full fixed-column event
objects in authoritative `event_id` order. All arguments are optional. Filters are exact matches,
and `after` is exclusive.

Persist `next_after`, not `last_event_id`, as the next scan cursor. A filtered page can contain no
matching events while still advancing `next_after` past unrelated history. The other page fields
are:

| field | meaning |
|---|---|
| `events` | Matching full event objects returned by this request. |
| `last_event_id` | Last matching event, or `None` when none matched. |
| `next_after` | Durable scan progress for the next request. |
| `snapshot_through_event_id` | Log frontier captured for this replay request. |

The M1 client returns one page per call; it does not provide `follow` or persistent cursor
storage.

## Query a published snapshot

```python
result = ground.query(
    "select source, count(*) as n "
    "from events group by source order by source, n",
    limit=10_000,
    timeout_secs=30,
)

print(result.columns)
for row in result.rows:
    print(row)

print(result.truncated)
print(result.receipt.as_of_event_id)
print(result.receipt.chain_head)
```

`query(sql, limit=None, timeout_secs=None)` maps to one JSON `POST /v1/query` request. The SDK
always requests the M1 JSON result form. The server accepts one confined read-only `SELECT` or
`WITH` statement and owns SQL validation, row limits, deterministic ordering requirements, and
timeouts.

`QueryResult` contains `columns`, array-shaped `rows`, the mandatory `truncated` flag, and a
`SnapshotReceipt`. Every relation read by one query comes from the warehouse generation named by
that receipt:

| receipt field | meaning |
|---|---|
| `as_of_event_id` | Published log frontier, or `None` for the empty generation. |
| `chain_head` | Integrity-chain head at that frontier. |
| `groundhog_version` | Binary version that published the generation. |
| `storage_schema_version` | Storage schema used by the generation. |
| `projections_hash` | Identity of the projection set. |

Replay reads the durable log immediately, while query reads the last published warehouse. Run
`groundhog project` or `groundhog rebuild` to publish newly ingested events. A live
`groundhog serve` process adopts a successful publication between requests.

## Read the catalog

```python
catalog = ground.catalog(source="stripe", stream="customers")

for item in catalog.streams:
    print(item["source"], item["stream"], item["event_count"])

print(catalog.receipt.as_of_event_id)
```

`catalog(source=None, stream=None)` maps to one `GET /v1/catalog` request. Both exact-match
filters are optional. `CatalogResult.streams` contains the published stream-level metadata, and
`CatalogResult.receipt` identifies the same coherent warehouse snapshot used to render it.

## Errors

All SDK exceptions derive from `GroundError` and retain the server's message. Server-originated
errors also expose `status` and the decoded JSON `body`.

| exception | meaning |
|---|---|
| `ValidationError` | Local request validation failed, or Groundhog returned HTTP 400/413. |
| `IdempotencyConflict` | HTTP 409: the same `(source, batch_id)` names different content. |
| `QueryError` | The query endpoint rejected or failed the statement, retaining its message. |
| `ConnectionError` | No complete response arrived within the configured attempts. |
| `GroundError` | Another server failure or an invalid server response. |

Per-event validation details are available through `ValidationError.errors`:

```python
from groundhog_sdk import ValidationError

try:
    ground.send("stripe", events, "stripe/page-1")
except ValidationError as error:
    for failure in error.errors:
        print(failure["index"], failure["error"])
```

An idempotency conflict and a validation failure are final. A lost response has an unknown
outcome, so the SDK resolves it only by retrying the identical buffered batch.

## Current M1 surface

The Python package currently exports:

```text
Ground
upserted, deleted, native
BatchReceipt, EventPage, QueryResult, CatalogResult, SnapshotReceipt
GroundError, ValidationError, IdempotencyConflict, QueryError, ConnectionError
PROTOCOL_VERSION
```

M1 intentionally does not include NDJSON streaming, server-local imports, automatic ingest-form
selection, `follow`, persistent cursor helpers, async parity, Arrow or DataFrame adapters,
connector management, or Groundhog installation and process supervision.
