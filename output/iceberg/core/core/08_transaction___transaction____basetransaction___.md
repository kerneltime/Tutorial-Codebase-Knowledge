# Chapter 8: Transaction (`Transaction`, `BaseTransaction`)

In [Chapter 7: Table Scan (`TableScan`, `BaseScan`, `TableScanContext`)](07_table_scan___tablescan____basescan____tablescancontext___.md), we learned how to precisely ask Iceberg for the data we want to read. But what about changing the table? What if you need to make several changes at once, and you need them all to succeed or fail together? For example, you might want to add new data files *and* update the table's description property simultaneously. If adding files succeeds but updating the property fails, your table might be in an inconsistent state.

This is where Iceberg **Transactions** come in. They enable you to group multiple operations into a single, atomic unit.

## The Problem: Keeping Changes Consistent

Imagine you're running an online store. When a customer places an order, you might need to:
1.  Add a new row to your `orders` table.
2.  Decrement the stock count in your `products` table.

What if step 1 succeeds, but step 2 fails (e.g., a network glitch)? You'd have an order for a product whose stock count wasn't updated – a data inconsistency!

Iceberg Transactions provide a "save point" mechanism. You can "stage" multiple changes (like adding data files, updating the table's schema, or changing its partitioning) within a transaction. When you're ready, you "commit" the transaction.
*   If the commit is successful, **all** your staged changes are applied together, atomically.
*   If any part of the transaction fails, or if you decide to roll it back, **none** of the changes are applied.

This ensures that your table (or tables, in more advanced scenarios) always remains in a consistent state.

Think of a transaction like a **shopping cart** before checkout:
*   You can add multiple items (operations like `newAppend()`, `updateSchema()`) to your cart.
*   Each item you add is "staged" within the transaction.
*   When you "checkout" (call `commitTransaction()`), either you pay for everything and get all your items (all operations succeed), or if your card is declined (an error occurs), you get none of the items, and the store's inventory is unaffected.

## Key Concepts of a Transaction

1.  **Staging Operations**: You start a transaction and then perform various table operations (like adding data, deleting data, changing the schema). These operations are not immediately applied to the live table. Instead, they are staged within the transaction.
2.  **Atomic Commit**: When you commit the transaction, Iceberg attempts to apply all staged changes as a single, indivisible operation. This relies on the atomic `commit()` capability of the underlying [Table Operations (`TableOperations`)](02_table_operations___tableoperations___.md) we discussed in Chapter 2.
3.  **Rollback (Implicit)**: If any part of the commit process fails, or if an operation within the transaction fails before the final commit, the transaction effectively rolls back. None of the staged changes will affect the table's visible state. Iceberg also handles cleaning up any temporary files created during failed operations.

## How to Use a Transaction

Let's walk through an example where we want to append a new data file to our `ordersTable` and also update its "comment" property. We want these two changes to be atomic.

```java
import org.apache.iceberg.Table;
import org.apache.iceberg.Transaction;
import org.apache.iceberg.DataFile;
// Assume 'ordersTable' is an initialized Iceberg Table object
// Assume 'newDataFile' is a DataFile object representing new data to add

// 1. Start a new transaction on the table
Transaction txn = ordersTable.newTransaction();
System.out.println("Started a new transaction.");

try {
    // 2. Stage an append operation within the transaction
    // This 'append' operation is part of the transaction
    txn.newAppend()
       .appendFile(newDataFile) // Add our new data file
       .commit(); // This "commits" the append TO THE TRANSACTION, not the live table yet.
                   // It updates the transaction's pending view of the table.
    System.out.println("Append operation staged in the transaction.");

    // 3. Stage an update to table properties within the transaction
    txn.updateProperties()
       .set("comment", "Appended new daily orders data.")
       .set("data-version", "1.1")
       .commit(); // This "commits" the property update TO THE TRANSACTION.
    System.out.println("Property update staged in the transaction.");

    // 4. Commit the entire transaction
    // This attempts to atomically apply all staged changes (append and properties)
    // to the actual ordersTable.
    txn.commitTransaction();
    System.out.println("Transaction committed successfully! Table updated.");

} catch (Exception e) {
    // If anything goes wrong during staging or the final commitTransaction,
    // none of the changes (append or property update) will be applied to the table.
    System.err.println("Transaction failed: " + e.getMessage());
    // In a real application, you might have explicit rollback or cleanup logic,
    // but Iceberg handles atomicity.
}
```

**What happens here?**

*   `ordersTable.newTransaction()`: Creates a transaction object linked to `ordersTable`.
*   `txn.newAppend().appendFile(newDataFile).commit()`:
    *   `txn.newAppend()`: Gives you an `AppendFiles` operation that knows it's part of `txn`.
    *   `.appendFile(newDataFile)`: Specifies the file to add.
    *   `.commit()`: This is crucial. It **stages** the append operation within the transaction. It doesn't write to the live table yet. Think of it as adding an item to your shopping cart. The transaction now internally knows about this pending append.
*   `txn.updateProperties().set(...).commit()`: Similarly, this stages the property changes within the transaction.
*   `txn.commitTransaction()`: This is the "checkout" moment. Iceberg takes all the staged changes (the appended file and the new properties) and attempts to apply them to `ordersTable` in one atomic step.
    *   **If successful**: `ordersTable` now has the new data file, and its "comment" and "data-version" properties are updated.
    *   **If it fails (e.g., due to a conflict or an issue writing metadata)**: `ordersTable` remains in the state it was in *before* the transaction started. The append and property changes are not applied.

## Under the Hood: `BaseTransaction` and its Magic

The `Transaction` interface is typically implemented by `org.apache.iceberg.BaseTransaction`. Here's a simplified idea of how it works:

1.  **Initialization**: When you call `table.newTransaction()`, a `BaseTransaction` object is created. It keeps track of:
    *   The original `TableOperations` of the table.
    *   The `TableMetadata` of the table when the transaction started (let's call this `baseMetadata`).
    *   A *copy* of this metadata that will be modified as operations are staged (let's call this `currentMetadataInTxn`). Initially, `currentMetadataInTxn` is the same as `baseMetadata`.
    *   A list to hold all the "pending updates" (like the append operation, the properties update, etc.).

2.  **Staging Operations**:
    *   When you call an operation like `txn.newAppend()`, it returns an operation object (e.g., `AppendFiles`). This operation object is special: it's linked to the transaction's internal `currentMetadataInTxn`.
    *   When you call `.commit()` on *that operation* (e.g., `appendOp.commit()`), the operation calculates what the `currentMetadataInTxn` would look like *if its changes were applied*. It then updates the transaction's `currentMetadataInTxn` to this new state. This all happens in memory within the transaction object. The real table metadata is not touched yet.

    ```mermaid
    sequenceDiagram
        participant UserApp as "Your Application"
        participant Txn as "Transaction Object"
        participant AppendOp as "Append Operation (within Txn)"
        participant TxnInternals as "Transaction Internals (currentMetadataInTxn)"

        UserApp->>Txn: newAppend()
        Txn-->>UserApp: AppendOp (configured for this Txn)
        UserApp->>AppendOp: appendFile(myFile)
        UserApp->>AppendOp: commit()
        AppendOp->>TxnInternals: Calculate new metadata with myFile
        TxnInternals->>TxnInternals: Update currentMetadataInTxn
        AppendOp-->>UserApp: Staging successful
    ```

3.  **Committing the Transaction**:
    *   When you call `txn.commitTransaction()`, `BaseTransaction` now has the final `currentMetadataInTxn` which reflects all staged changes.
    *   It then takes this `currentMetadataInTxn` and attempts to commit it to the actual table using the table's original `TableOperations`. This is where the real atomic write to the metastore happens.
    *   **Optimistic Concurrency**: Before committing, Iceberg usually refreshes the table's metadata from the store. If the table has changed since the transaction began (i.e., `baseMetadata` is no longer the live table's current metadata), Iceberg will try to re-apply all the staged changes on top of the *newly refreshed* metadata. If this re-application is successful, it proceeds with the commit. If there's a conflict that can't be resolved, the commit might fail. This is a form of optimistic locking.

    ```mermaid
    sequenceDiagram
        participant UserApp as "Your Application"
        participant Txn as "BaseTransaction"
        participant RealTableOps as "Table's Real TableOperations"
        participant Metastore as "Actual Metastore"

        UserApp->>Txn: commitTransaction()
        Note over Txn: Has final 'currentMetadataInTxn' with all staged changes.
        Txn->>RealTableOps: refresh() (get latest actual table metadata)
        RealTableOps-->>Txn: actualBaseMetadata
        alt If actualBaseMetadata differs from Txn's initial base
            Txn->>Txn: Re-apply staged changes on top of actualBaseMetadata
        end
        Txn->>RealTableOps: commit(actualBaseMetadata, currentMetadataInTxn)
        RealTableOps->>Metastore: Atomically update metadata pointer
        Metastore-->>RealTableOps: Success/Failure
        RealTableOps-->>Txn: Commit result
        Txn-->>UserApp: Transaction Succeeded/Failed
    ```

### `BaseTransaction.java` Snippets

Let's look at simplified pieces from `src/main/java/org/apache/iceberg/BaseTransaction.java`:

```java
// Simplified structure of BaseTransaction
public class BaseTransaction implements Transaction {
    private final String tableName;
    private final TableOperations ops; // The real TableOperations for the actual table
    private final TransactionTable transactionTable; // An inner table view for the transaction
    private final TableOperations transactionOps; // Special Ops for the transactionTable
    private final List<PendingUpdate> updates; // List of staged operations
    private TableMetadata base; // Metadata when transaction started
    private TableMetadata current; // Metadata with staged changes, within the transaction

    BaseTransaction(String tableName, TableOperations ops, /*...other params...*/) {
        this.tableName = tableName;
        this.ops = ops;
        this.base = ops.refresh(); // Get current metadata from the table
        this.current = this.base;  // Initially, transaction's view is same as base
        this.updates = Lists.newArrayList();
        this.transactionOps = new TransactionTableOperations(); // Inner class
        this.transactionTable = new TransactionTable();       // Inner class
        // ...
    }

    // When you call txn.newAppend(), txn.updateSchema(), etc.
    // these methods create a PendingUpdate object (like AppendFiles, SchemaUpdate)
    // and add it to the 'updates' list.
    // For example:
    @Override
    public AppendFiles newAppend() {
        // ... (check last operation committed state) ...
        // 'transactionOps' ensures changes are applied to 'this.current' metadata
        AppendFiles append = new MergeAppend(tableName, transactionOps);
        updates.add(append);
        return append;
    }

    // The core of the final commit
    @Override
    public void commitTransaction() {
        // ... (precondition checks) ...
        if (base == current) { // No changes staged
            return;
        }

        // Simplified: Attempt to commit 'current' metadata (with all staged changes)
        // using the 'ops' (real TableOperations).
        // This involves retry logic and re-applying updates if 'base' has changed.
        Tasks.foreach(ops)
            .retry(/*...config...*/)
            .run(underlyingOps -> {
                // If underlying table changed, re-apply updates on new base
                if (base != underlyingOps.refresh()) {
                    this.base = underlyingOps.current();
                    this.current = underlyingOps.current(); // Reset txn's current
                    for (PendingUpdate update : updates) {
                        update.commit(); // Re-commit each staged update to txn's new 'current'
                    }
                }
                // Perform the actual atomic commit to the metastore
                underlyingOps.commit(base, current);
            });
        // ... (cleanup of temporary files) ...
    }

    // Inner class that acts as the TableOperations for the TransactionTable
    // Its commit method updates the 'current' metadata within BaseTransaction
    public class TransactionTableOperations implements TableOperations {
        @Override
        public TableMetadata current() { return BaseTransaction.this.current; }

        @Override
        public void commit(TableMetadata txBase, TableMetadata metadataToStage) {
            // txBase is BaseTransaction.this.current before this specific update
            // metadataToStage includes changes from this specific update
            if (txBase != BaseTransaction.this.current) {
                throw new CommitFailedException("Transaction state changed");
            }
            BaseTransaction.this.current = metadataToStage; // Update transaction's view
            // ... (mark that this specific operation within the txn has "committed")
        }
        // ... other methods delegate or use a temporary ops instance ...
    }
    // ... TransactionTable inner class uses TransactionTableOperations ...
}
```
The key is that operations like `append.commit()` inside a transaction modify an *in-memory* `TableMetadata` object held by the `BaseTransaction`. Only `transaction.commitTransaction()` attempts to make these changes permanent in the underlying table storage.

### `Transactions.java` Utility

Iceberg provides a utility class `org.apache.iceberg.Transactions` for creating transactions, especially for scenarios like creating or replacing tables, which have slightly different starting conditions.

```java
// From src/main/java/org/apache/iceberg/Transactions.java
public final class Transactions {
    private Transactions() {}

    // For creating a new table
    public static Transaction createTableTransaction(
            String tableName, TableOperations ops, TableMetadata startMetadata) {
        // 'startMetadata' is the initial metadata for the new table
        return new BaseTransaction(tableName, ops, TransactionType.CREATE_TABLE, startMetadata);
    }

    // For typical updates to an existing table
    public static Transaction newTransaction(String tableName, TableOperations ops) {
        // ops.refresh() gets the latest metadata to start the transaction from
        return new BaseTransaction(tableName, ops, TransactionType.SIMPLE, ops.refresh());
    }
    // ... other methods for replaceTableTransaction etc. ...
}
```
Most often, you'll use `table.newTransaction()`, which internally likely calls something similar to `Transactions.newTransaction(table.name(), table.operations())`.

### `CommitCallbackTransaction.java`

Sometimes, you might want to perform an action *after* an Iceberg transaction has successfully committed. For this, Iceberg provides `CommitCallbackTransaction`. It's a wrapper around another transaction.

```java
// From src/main/java/org/apache/iceberg/CommitCallbackTransaction.java
class CommitCallbackTransaction implements Transaction {
    private final Transaction wrapped; // The actual transaction
    private final Runnable callback;   // What to run after successful commit

    // ... constructor ...

    @Override
    public void commitTransaction() {
        wrapped.commitTransaction(); // Delegate to the actual transaction
        callback.run();              // If successful, run the callback
    }
    // ... all other Transaction methods simply delegate to 'wrapped' ...
}
```
You could use this to, for example, send a notification or update an external system once your Iceberg changes are confirmed.

## Conclusion

Transactions are a powerful feature in Iceberg for ensuring data consistency when making multiple changes to a table. They allow you to stage several operations and then commit them atomically – all or nothing.

Key Takeaways:
*   A **Transaction** groups multiple table modifications into a single atomic unit.
*   Operations are **staged** within the transaction and only applied to the live table upon `transaction.commitTransaction()`.
*   If any part fails, none of the staged changes are applied, maintaining table consistency.
*   `BaseTransaction` is the common implementation, managing a list of pending updates and an in-progress version of `TableMetadata`.
*   The final commit uses the underlying table's `TableOperations` for the atomic update.
*   Helper classes like `Transactions` assist in creating transactions for different scenarios.

Now that you understand how to read data with scans and modify tables safely with transactions, you might wonder about the very foundation: how does Iceberg actually read and write the physical files (metadata files, manifest files, data files) on storage systems like HDFS or S3? That's the role of the `FileIO` abstraction.

Next up, let's explore [Chapter 9: FileIO (`FileIO`)](09_fileio___fileio___.md).

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)