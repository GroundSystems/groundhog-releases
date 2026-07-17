---
title: Verification and recovery
description: Verify a deployment and respond safely to startup, storage, and integrity failures.
---

## Name

`groundhog-recovery`: verification depth, crash recovery, and operator response.

## Principles

Recovery never guesses. The binary opens a data directory only when it can identify one
authoritative durable history. Ambiguous or contradictory state is refused.

Read-only commands do not repair data. Opening the directory for writing may complete recognized
interrupted final writes before accepting new mutation. Verification is always read-only.

## Ordinary open analysis

Every open checks enough stored state to establish:

- a compatible storage format;
- one coherent committed history frontier;
- strictly ordered event ranges; and
- consistent durable `(source, batch_id)` commitments.

If an interrupted final write can be safely excluded or completed, a writer open recovers it
before serving. Conflicting complete records or representations are not silently discarded.

## Structural verification

`groundhog verify` performs a thorough stored-history check:

- SHA-256 of each authoritative segment file;
- actual Parquet row count and exact first/last event-ID range;
- strict row order within each segment;
- exact partition of segment rows by recorded batches;
- recomputation of batch digests from stored event commitments; and
- the corresponding pending/tail facts from the captured inventory.

It also reports recognized orphans and repairable remnants without changing them.

## Chain verification

`groundhog verify --chain` additionally recomputes, for every event:

1. `content_hash` from canonical payload JSON;
2. `event_hash` from the canonical fixed envelope; and
3. the ordered logical chain from the fixed genesis head.

It compares chain heads at required boundaries and at the captured frontier. This detects payload
or envelope alteration, insertion, removal, or reordering inside the available history.

Local chain consistency does not prove that someone controlling the entire directory did not
replace it coherently. External signed, mirrored, and witnessed anchors are not supported.

## Reports and exit status

On conclusive verification success or failure, parse the standard-output JSON report. Exit 0
means requested checks passed; exit 3 means a stable failure code identifies the first conclusive
violation.

Exit 1 means the operation could not establish a conclusive verification result because of an
operational error. Exit 2 is invalid arguments/configuration. Exit 4 is a recognized but
unsupported compatibility, security, or anchor requirement.

Do not collapse exits 1, 3, and 4 into one “bad log” state; they require different operator
responses.

## Orphans and remnants

An **orphan** is recognized non-authoritative storage, such as an unreferenced segment or private
temporary. It is inventoried but not verified.

A **remnant** is excluded incomplete final state that a compatible writer recovery may repair.

Either list may be non-empty while verification succeeds. Their classification is evidence, not
permission for ad hoc deletion. `verify --clean` is not available.

## Writer poisoning

If a write fails in a way that leaves its outcome uncertain, the active writer stops accepting
mutation:

- the triggering HTTP request fails or loses a definitive result;
- later mutation requests return 503;
- no further append is accepted through that writer; and
- the service must close and reopen the directory through normal recovery.

Clients resolve the original batch outcome only after reopen by retrying identical content under
the same idempotency key. Recovery either finds the committed batch and returns `duplicate`, or
finds no commitment and appends it once.

## Operator runbook: verification failure

When `verify` exits 3:

1. stop mutation and preserve the JSON report plus standard-error diagnostics;
2. stop the live writer cleanly if it is still running;
3. preserve a filesystem-level copy of the affected directory before experimentation;
4. record the binary build identity and configuration used;
5. do not edit storage files, delete orphans, or rebuild over the only copy;
6. classify the stable failure code and compare with a known coherent backup; and
7. restore or investigate on an isolated copy.

`rebuild` is not a repair for a corrupt log. It repairs only the disposable warehouse and depends
on a readable authoritative log.

## Operator runbook: crash or poisoned writer

After a crash or HTTP 503 writer-unavailable state:

1. stop the old process and confirm it no longer owns the socket/writer;
2. restart `serve` with the same configuration, allowing writer recovery to run;
3. probe a routed endpoint;
4. retry any ambiguous ingest with identical `(source, batch_id, content)`; and
5. run `verify`, escalating to `verify --chain` when the event requires deeper evidence.

Do not remove `.lock`; confirm the old process has exited and let the binary reopen the directory.

## Operator runbook: invalid warehouse

If the log verifies but query/catalog cannot open the disposable warehouse:

```sh
groundhog rebuild --config /path/to/groundhog.toml
```

The replacement is atomically published. If a service already holds a valid old generation, it
may remain live and adopts the replacement between requests. If service startup
itself cannot open the only warehouse, rebuild first, then start it.

## Restore validation

A restored directory is not proven merely because files exist. In an isolated location:

```sh
groundhog verify --config ./restored/groundhog.toml
groundhog verify --chain --config ./restored/groundhog.toml
groundhog rebuild --config ./restored/groundhog.toml
```

Then serve it on an isolated socket and compare known query results and receipts. Keep the restore
test from contacting production clients or sharing a writer path.

## See also

[`verify`](/commands#verify),
[`rebuild`](/commands#rebuild),
[storage](/concepts/storage),
[deployment operations](/operations/deployment)
