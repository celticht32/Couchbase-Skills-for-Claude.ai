# When to use transactions

## The decision ladder

Before reaching for a transaction, check whether a cheaper primitive covers the need:

| Tool | Scope | Guarantee | Cost |
|---|---|---|---|
| Single KV write (`upsert`) | 1 document | Atomic write of the document | 1× |
| Subdocument op (`mutate_in`) | 1 document, multiple fields | Atomic multi-field update | 1× |
| CAS-based optimistic lock | 1 document | Prevents lost writes | 1× + retry on conflict |
| **Distributed Transaction** | Multiple documents | All-or-nothing atomicity | **3-5×** |

**If the operation touches only one document:** use a KV write or `mutate_in`. No transaction needed.

**If the operation touches two documents but only one needs a consistent view:** consider modeling as one document (embed the second document's relevant fields). Often cheaper than a transaction.

**If you need true all-or-nothing across two or more documents:** use a transaction.

## Good transaction use cases

**Financial transfers:** debit one account, credit another — both must succeed or neither.
```
BEGIN
  debit account A by $100
  credit account B by $100
COMMIT
```

**Inventory + order:** reserve inventory and create an order atomically.
```
BEGIN
  decrement inventory.available_quantity
  insert order document
COMMIT
```

**Multi-document state machine:** advance a workflow that touches a status document and a history document.

**Reference integrity:** create a parent document and its first child atomically.

## Poor transaction use cases (and better alternatives)

**Incrementing a counter across documents:** use sub-document `increment` on a single document instead. It's atomic without a transaction.

**Updating all documents matching a query:** transactions don't work on query results. Process each document independently, accept eventual consistency, or use Eventing for fan-out.

**Logging an event while updating a record:** log writes are append-only; they don't need to be atomic with the record update. Write them independently.

**Idempotent operations:** if rerunning the same operation twice produces the same result (idempotent), a simple write with a CAS check handles retries without full transaction overhead.

## Modeling to avoid transactions

The most effective way to reduce transaction use is document design. Common patterns:

**Embed instead of reference:** if order items are always created/deleted with their order, embed them in the order document. No transaction needed to keep them consistent.

**Event sourcing:** instead of updating a balance in place, append an event (`{type: "credit", amount: 100, timestamp: ...}`). The balance is derived from the event log. No transaction needed for the append; consistency is eventual.

**Saga pattern:** break a multi-step workflow into independent idempotent steps. Each step writes its own status. A coordinator (Eventing function or application) drives the steps. On failure, run compensating actions. More complex than a transaction but scales better.

**Single document with sub-documents:** combine two logically related things into one document if they always change together. Example: order + shipping address embedded, not referenced.
