---
title: Warehouse and query freshness
description: Understand publication, queryable relations, catalog data, and SQL freshness.
---

The warehouse is Groundhog's published SQL view of event history. Clients query it through
`POST /v1/query` and discover its contents through `GET /v1/catalog`.

The event log remains the durable history. The warehouse can be recreated with `rebuild`.

## Replay and query have different freshness

Ingested events are available to replay immediately. SQL and catalog requests use the most recent
warehouse publication.

| interface | data visible |
|---|---|
| `GET /v1/events` | Committed events through the replay request's captured frontier. |
| `POST /v1/query` | Events through the last successful `project` or `rebuild`. |
| `GET /v1/catalog` | Metadata from that same published warehouse. |

Run `project` when SQL and catalog should include newly ingested events. A running service adopts
the new publication without restarting. Requests already in progress continue against the
snapshot with which they began.

Every query and catalog response includes a receipt identifying the exact published frontier.

## Queryable relations

### `events`

One row per published event:

| column | SQL type |
|---|---|
| `event_id` | `UUID` |
| `source` | `VARCHAR` |
| `stream` | `VARCHAR` |
| `record_key` | `VARCHAR` |
| `kind` | `VARCHAR` |
| `occurred_at` | nullable `TIMESTAMP` |
| `observed_at` | `TIMESTAMP` |
| `payload` | `JSON` |
| `content_hash` | `VARCHAR` |
| `batch_id` | `VARCHAR` |
| `event_hash` | `VARCHAR` |

Rows have no implicit order. Use `ORDER BY event_id` for authoritative event order.

### `meta.streams`

One row per published source and stream:

| column | meaning |
|---|---|
| `source` | Source name. |
| `stream` | Stream name. |
| `event_count` | Published event count. |
| `first_observed_at` | Earliest published observation time. |
| `last_observed_at` | Latest published observation time. |
| `kinds` | Sorted distinct event kinds. |

This relation is also exposed through the catalog endpoint.

### `meta.projection_state`

One row describing the published snapshot:

| column | meaning |
|---|---|
| `last_event_id` | Published event frontier; null for an empty history. |
| `chain_head` | Integrity commitment through that frontier. |
| `groundhog_version` | Binary version that published it. |
| `storage_schema_version` | Storage version used. |
| `projections_hash` | Identity of the projection set used. |

HTTP receipts call `last_event_id` `as_of_event_id`.

## Catalog

`GET /v1/catalog` lists published source/stream pairs, event counts, observation times, and event
kinds. Narrow it with exact `source` and `stream` parameters:

```sh
curl --unix-socket data/ground.sock \
  'http://ground/v1/catalog?source=stripe&stream=customers'
```

Use the catalog to discover available data before writing SQL.

## Query behavior

`POST /v1/query` accepts one read-only `SELECT` or `WITH` statement and returns JSON rows.

Queries can read only published Groundhog relations. File and network access, database writes,
multiple statements, extensions, and nondeterministic functions are rejected.

Row limits and timeouts come from `groundhog.toml`. A result cut by the server row limit needs
a total top-level order. Ordering by every selected column or using `ORDER BY ALL` satisfies the
general case.

## Receipts

Successful query and catalog responses include:

```json
{
  "as_of_event_id": "...",
  "chain_head": "...",
  "groundhog_version": "0.1.0",
  "storage_schema_version": 1,
  "projections_hash": "..."
}
```

Retain the receipt with reports, exports, or automated outputs when you need to identify which
history produced them. It identifies the published warehouse snapshot, which may be behind the
current replay frontier.

## Rebuild

`rebuild` recreates the warehouse from durable history. It can replace missing, stale, or invalid
query data and may run while `serve` is active.

Do not write directly to `warehouse.duckdb`; Groundhog may replace it during publication.

## Current limitations

The warehouse does not publish `current.*` state relations, inferred payload columns, custom
projections, Arrow results, or automatic background refresh.

## See also

[`project`](/commands#project),
[`rebuild`](/commands#rebuild),
[HTTP API](/references/http-api),
[Events](/concepts/events)
