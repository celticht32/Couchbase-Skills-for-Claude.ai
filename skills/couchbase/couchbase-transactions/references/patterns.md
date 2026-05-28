# Transaction patterns

## Python

```python
from couchbase.cluster import Cluster
from couchbase.exceptions import TransactionFailed, TransactionExpired

def transfer_funds(cluster, from_id, to_id, amount):
    accounts = cluster.bucket("bank").scope("_default").collection("accounts")

    def txn_logic(ctx):
        # Read both documents inside the transaction
        from_doc = ctx.get(accounts, from_id)
        to_doc = ctx.get(accounts, to_id)

        from_data = from_doc.content_as[dict]
        to_data = to_doc.content_as[dict]

        if from_data["balance"] < amount:
            raise ValueError(f"Insufficient funds: {from_data['balance']} < {amount}")

        # Replace both documents
        from_data["balance"] -= amount
        to_data["balance"] += amount

        ctx.replace(from_doc, from_data)
        ctx.replace(to_doc, to_data)
        # ctx.insert(audit_coll, str(uuid.uuid4()), audit_record)  # can include more ops

    try:
        result = cluster.transactions.run(txn_logic)
        return result.transaction_id
    except TransactionExpired as e:
        raise RuntimeError(f"Transaction timed out: {e}")
    except TransactionFailed as e:
        raise RuntimeError(f"Transaction failed after retries: {e}")
```

## Java

```java
TransactionResult result = cluster.transactions().run(ctx -> {
    TransactionGetResult fromDoc = ctx.get(accounts, "account::" + fromId);
    TransactionGetResult toDoc = ctx.get(accounts, "account::" + toId);

    JsonObject from = fromDoc.contentAsObject();
    JsonObject to = toDoc.contentAsObject();

    double balance = from.getDouble("balance");
    if (balance < amount) {
        throw new InsufficientFundsException("Balance " + balance + " < " + amount);
    }

    from.put("balance", balance - amount);
    to.put("balance", to.getDouble("balance") + amount);

    ctx.replace(fromDoc, from);
    ctx.replace(toDoc, to);
});
```

## Node.js

```javascript
const result = await cluster.transactions().run(async (ctx) => {
    const fromDoc = await ctx.get(accounts, `account::${fromId}`);
    const toDoc = await ctx.get(accounts, `account::${toId}`);

    const from = fromDoc.content;
    const to = toDoc.content;

    if (from.balance < amount) {
        throw new Error(`Insufficient funds`);
    }

    from.balance -= amount;
    to.balance += amount;

    await ctx.replace(fromDoc, from);
    await ctx.replace(toDoc, to);
});
```

## Error handling

```python
from couchbase.exceptions import (
    TransactionFailed,
    TransactionExpired,
    TransactionCommitAmbiguous,
    DocumentNotFoundException,
)

try:
    cluster.transactions.run(txn_logic)

except TransactionCommitAmbiguous:
    # The commit was sent but we don't know if it succeeded.
    # The SDK retried until timeout. You must check the database state
    # to determine if the transaction committed. Do NOT retry automatically —
    # you may double-apply the operation.
    check_and_reconcile()

except TransactionExpired:
    # Transaction did not commit within the timeout window.
    # Safe to retry from the application side.
    raise RetryableError("Transaction timed out, please retry")

except TransactionFailed as e:
    # Transaction failed for a non-retryable reason (e.g. business logic exception
    # raised inside the lambda, or document not found).
    # Do NOT retry — the error is deterministic.
    raise e
```

**`TransactionCommitAmbiguous` is the tricky one.** It means the SDK sent the commit but the response was lost (network issue, node restart). The transaction may or may not have committed. You must check the database to determine the truth — there's no safe automatic retry.

Design for this by including an idempotency key in your transaction. Before the transaction starts, check if the idempotency key already exists; if it does, the transaction already committed.

## Idempotency key pattern

```python
import uuid

def safe_transfer(cluster, from_id, to_id, amount, idempotency_key=None):
    if idempotency_key is None:
        idempotency_key = str(uuid.uuid4())

    txn_log = cluster.bucket("bank").scope("_default").collection("txn_log")
    accounts = cluster.bucket("bank").scope("_default").collection("accounts")

    def txn_logic(ctx):
        # Check idempotency key — if it exists, transaction already ran
        try:
            ctx.get(txn_log, idempotency_key)
            return  # Already committed, nothing to do
        except DocumentNotFoundException:
            pass  # Not committed yet, proceed

        from_doc = ctx.get(accounts, from_id)
        to_doc = ctx.get(accounts, to_id)

        from_data = from_doc.content_as[dict]
        to_data = to_doc.content_as[dict]

        if from_data["balance"] < amount:
            raise ValueError("Insufficient funds")

        from_data["balance"] -= amount
        to_data["balance"] += amount

        ctx.replace(from_doc, from_data)
        ctx.replace(to_doc, to_data)
        ctx.insert(txn_log, idempotency_key, {
            "from_id": from_id, "to_id": to_id,
            "amount": amount, "committed_at": datetime.utcnow().isoformat()
        })

    cluster.transactions.run(txn_logic)
    return idempotency_key
```

## Configuration tuning

```python
from couchbase.transactions import TransactionConfig
from datetime import timedelta

txn_config = TransactionConfig(
    expiration_time=timedelta(seconds=30),  # max transaction lifetime (default 15s)
    cleanup_window=timedelta(seconds=60),   # how often to clean up lost transactions
)

cluster = Cluster(connection_string, ClusterOptions(
    PasswordAuthenticator(username, password),
    transaction_config=txn_config
))
```

Increase `expiration_time` only if your transaction genuinely needs more time. Long-lived transactions hold locks and block concurrent writers.
