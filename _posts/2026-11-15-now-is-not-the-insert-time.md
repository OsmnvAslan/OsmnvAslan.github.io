---
layout: post
title: "now() is not the insert time"
date: 2026-11-15
description: How a DEFAULT now() column let an outbox relay publish two events for one key in the wrong order.
---

[txoutbox](https://github.com/OsmnvAslan/txoutbox) 0.2.3 fixed a bug where the Postgres
adapter could publish two messages with the same key in the wrong order. The feature that
allowed it was the one meant to prevent reordering. This is how it happened.

## The feature

An outbox table gets its ids from a sequence, and a sequence hands out values at `INSERT`,
not at `COMMIT`. A transaction that took id 1 can commit after a transaction that took id 2.
A relay that reads in id order sees id 2 first, publishes it, and only later finds id 1.

`PostgresStorage(visibility_delay=...)` exists for this. The relay only claims rows older
than the delay. If the delay is longer than your slowest producing transaction, every lower
id has committed by the time a higher one becomes visible.

"Older" was measured from `created_at`, and the schema declared it like this:

```sql
created_at TIMESTAMPTZ NOT NULL DEFAULT now()
```

## The bug

In Postgres `now()` is the start time of the current transaction, not the current time.
Every row a transaction inserts gets the same `created_at`, however long the transaction
has been running.

So a lower id can look younger than a higher one. Take a delay of 2 seconds:

| Time | Transaction X | Transaction Y |
| --- | --- | --- |
| 0.0 s | `BEGIN` | |
| 3.0 s | | `BEGIN`, insert id 1 (`created_at` = 3.0) |
| 4.0 s | insert id 2 (`created_at` = 0.0) | |
| 4.5 s | `COMMIT` | `COMMIT` |

At 4.5 s the relay asks for rows older than 2 seconds. Id 2 claims to be 4.5 seconds old
and passes. Id 1 is 1.5 seconds old and does not. Both rows have the same key, and id 2
goes out first.

Strict ordering did not help. Its predicate held a row back only while an earlier row with
the same key was leased or waiting for a retry. An earlier row that was merely too young
was not a reason to wait.

## The fix

Two changes, and both are needed.

The schema now uses the real insert time:

```sql
created_at TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp()
```

And with strict ordering, the claim query holds a row back while an earlier pending row
with its key is still inside the delay (simplified):

```sql
AND NOT EXISTS (
  SELECT 1 FROM outbox p
  WHERE p.key = o.key AND p.id < o.id AND p.status = 'pending'
    AND (p.retry_at > now() OR p.lease_until > now()
         OR p.created_at > now() - $4::interval)
)
```

The predicate alone is not enough for old tables. A row whose transaction has not committed
is invisible to the check, so with the `now()` default the delay would have to exceed twice
your longest producer transaction. `create_schema()` now switches the default on existing
tables; it is a metadata-only `ALTER TABLE`.

## The test

The regression test inserts a young row and then an older row with the same key, the older
one backdated by five seconds:

```python
assert list(await storage.claim(batch_size=10, lease=LEASE, worker_id="w")) == []
await asyncio.sleep(1.1)
got = await storage.claim(batch_size=10, lease=LEASE, worker_id="w")
assert [m.payload for m in got] == [b"young", b"old"]
```

Before the fix the first claim returned the old row.

## What I took from it

A timestamp column answers one question, and it is worth writing down which. "When did this
transaction start" and "when was this row written" are different facts, and `DEFAULT now()`
silently picks the first. If anything downstream compares the column with the wall clock,
it wants `clock_timestamp()`.
