# Transaction mechanics

## Two-phase commit overview

Couchbase transactions use a client-coordinated two-phase commit:

```
Phase 1 — Staging
  For each document to be written:
    1. Write a "staged" version of the document to a special transaction metadata section
    2. Mark the ATR entry as "pending"

Phase 2 — Commit
  1. Mark the ATR entry as "committed" (this is the durable commit point)
  2. For each document: move the staged version to the live document slot
  3. Remove the staged metadata from each document
  4. Mark the ATR entry as "complete"
```

If the client crashes after Phase 2 step 1 (ATR committed) but before cleanup, another client or the lost transaction cleanup job completes the commit. Documents are never left in an inconsistent state.

If the client crashes before the ATR is committed, the transaction is rolled back by the cleanup process.

## Active Transaction Records (ATRs)

ATRs are special documents in the same bucket/collection as your data. They track in-flight transactions. You'll see them in the bucket as keys like `_txn:atr-0`, `_txn:atr-1`, ..., `_txn:atr-1023`.

**Key facts:**
- 1024 ATR vBucket slots per collection by default
- Each transaction is assigned to one ATR slot based on its transaction ID
- ATR documents are small and cleaned up after commit/rollback
- ATR documents count against bucket quota — negligible in practice

**You should not:**
- Delete ATR documents manually — this can cause transaction loss
- Run queries on `_txn:` keys in production loops (they're transient)

If you see persistent `_txn:atr-*` documents that aren't cleaning up, you likely have orphaned transactions from crashed clients. The SDK's background cleanup process handles these, but it runs on a configurable interval.

## Retry and expiry

Transactions retry automatically on transient failures (write conflicts, temporary node unavailability). The retry loop runs for up to `transaction_timeout` seconds (default: 15 seconds), using exponential backoff.

```
Transaction starts
  → write conflict on document A
  → SDK waits (backoff)
  → retries from the beginning of the lambda
  → succeeds on second attempt
  → commits
```

Your transaction lambda must be idempotent within a single transaction attempt — it may run multiple times. Don't generate UUIDs or timestamps outside the lambda; generate them inside so they're consistent across retries.

```python
# WRONG — UUID generated once, but lambda may retry
new_id = str(uuid.uuid4())
def txn_logic(ctx):
    ctx.insert(collection, new_id, {...})

# CORRECT — UUID generated inside lambda, consistent per attempt
def txn_logic(ctx):
    new_id = str(uuid.uuid4())
    ctx.insert(collection, new_id, {...})
```

**`transaction_timeout`:** if a transaction doesn't commit within this window, it rolls back and raises `TransactionExpiredException`. For long-running transactions, increase this — but also question whether a transaction spanning seconds is the right design.

## Isolation level

Couchbase transactions provide **Read Committed** isolation by default:
- Reads within a transaction see committed data at the point the read is issued
- Other transactions cannot see your uncommitted writes
- You can see your own uncommitted writes within the same transaction

There is no Serializable isolation. For Serializable semantics, use CAS-based locking or design out the conflict possibility.

## Performance characteristics

| Operation | Latency multiplier vs KV |
|---|---|
| Simple 1-document transaction | ~3× |
| 2-document transaction | ~4× |
| 5-document transaction | ~5-7× |
| Transaction with write conflict + retry | ~8-10× |

Most of the overhead is from ATR writes (two of them: staging and commit point). High-latency storage (cloud block storage, cross-zone Capella) amplifies this further.

For write-heavy workloads, profile transaction throughput against your requirements before committing to the design.

## Lost write protection

Couchbase transactions use CAS (Compare-And-Swap) internally. If two transactions try to modify the same document simultaneously, one will detect the conflict and retry. There is no "last write wins" — conflicts are detected and retried deterministically.
