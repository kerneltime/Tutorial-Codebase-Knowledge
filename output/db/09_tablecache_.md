# Chapter 9: TableCache

In [Chapter 8: DBCheckpoint / RDBCheckpointManager](08_dbcheckpoint___rdbcheckpointmanager_.md), we learned how to create complete snapshots (checkpoints) of our database, which are essential for backups and recovery. Checkpoints help ensure data safety.

Now, let's shift our focus to **performance**. Accessing data directly from disk storage, even fast SSDs, is much slower than accessing data already loaded into the computer's main memory (RAM). If our application frequently reads the same pieces of data over and over again, going to the disk every single time can become a bottleneck. How can we speed this up?

## The Problem: Waiting for the Filing Cabinet

Imagine your database is like a large filing cabinet ([Table](05_table_.md) stored on disk). Every time you need a specific file (a key-value pair), you have to walk over to the cabinet, find the right drawer, locate the file, and read it. This takes time.

Now, suppose you constantly refer to the same handful of files throughout the day – maybe the "Active Projects" list or the "Top Customers" file. Wouldn't it be much faster if you kept copies of *those specific files* right on your desk in a small tray? That way, you could grab them instantly without going back to the main cabinet every time.

This is the core idea behind caching in our `db` library.

## The Solution: `TableCache` - Your Desk Tray for Data

A `TableCache` is an **in-memory cache** designed to speed up access to frequently used data within a specific `Table`. It acts like that small desk tray, holding copies of data that was recently read from or written to the main disk-based table (the filing cabinet).

Here's how it works:

1.  **Sits in Between:** The `TableCache` lives between your application code (when it interacts with a `Table` object) and the actual disk storage for that table.
2.  **Memory Speed:** It stores copies of some key-value pairs directly in the computer's fast RAM.
3.  **Check the Cache First:** When you try to `get` data from a `Table` that has a cache enabled, the system checks the `TableCache` *first*.
    *   **Cache Hit:** If the data you requested is found in the cache (it's in the desk tray), it's returned immediately – super fast!
    *   **Cache Miss:** If the data is *not* in the cache, the system then goes to the disk (the filing cabinet) to retrieve it. Optionally, it might then place a copy of that data into the cache for potential future use.
4.  **Updates:** When you `put` or `delete` data, the cache is also updated accordingly to keep it consistent with the intended state.

Think of it as an intelligent layer that tries to anticipate what data you'll need and keeps it ready in memory for quick access.

### Cache Keys and Values

Internally, the cache needs a way to wrap the keys and values from your table.

*   `CacheKey<KEY>`: A simple wrapper around the actual key object (like a `String` username).
*   `CacheValue<VALUE>`: A wrapper around the actual value object (like a `String` email). It also contains important metadata, like an **epoch** number. The epoch usually corresponds to a transaction or operation sequence number. This helps the cache manage its entries and know when certain cached items might be outdated or safe to remove after the corresponding data has been permanently saved to disk.

You usually don't interact with `CacheKey` and `CacheValue` directly; they are used internally by the `Table` and `TableCache` implementations.

### Different Caching Strategies

Not all tables benefit from caching in the same way. The `db` library offers different `TableCache` implementations (strategies):

1.  **`FullTableCache<KEY, VALUE>`:**
    *   **Analogy:** Keeping a complete photocopy of the *entire* filing drawer on your desk.
    *   **What it does:** Tries to keep *all* key-value pairs from the table loaded in memory.
    *   **Best for:** Relatively small tables that are read very frequently. If the whole table fits comfortably in memory, this provides the fastest possible access, as nearly every read becomes a cache hit.
    *   **Trade-off:** Can consume significant memory if the table is large.

2.  **`PartialTableCache<KEY, VALUE>`:**
    *   **Analogy:** Keeping only the *active* files you've recently used or modified on your desk tray.
    *   **What it does:** Caches only the items that have been recently accessed or modified (put/deleted). Older or less frequently used items might not be in the cache.
    *   **Best for:** Larger tables where you frequently access a smaller subset of the data, or tables with many writes. It balances performance improvement with memory usage.
    *   **Trade-off:** Reads for data not currently in the cache will still need to go to disk (cache miss).

3.  **`TableNoCache<KEY, VALUE>`:**
    *   **Analogy:** Not using a desk tray at all; always going straight to the filing cabinet.
    *   **What it does:** Provides a cache interface but doesn't actually store anything. Every operation goes directly to the disk.
    *   **Best for:** Tables that are very large and infrequently accessed, or where the overhead of caching isn't worth the benefit.

### Cache Cleanup (Eviction)

Caches store data in memory, and memory is a finite resource. We can't just keep adding items to the cache forever. Eventually, old or less relevant entries need to be removed (evicted) to make space or to reflect that the underlying data has been permanently saved to disk and the cached copy is no longer strictly necessary.

The `TableCache` implementations often use the **epoch** number associated with `CacheValue`s for cleanup. When the system knows that all operations up to a certain epoch have been safely flushed to disk (e.g., after a [DBCheckpoint](08_dbcheckpoint___rdbcheckpointmanager_.md) or a successful data flush), it can tell the cache to `cleanup` entries associated with those old epochs.

*   In `FullTableCache`, cleanup mainly removes entries that were marked for deletion (`CacheValue` with a null actual value) once the deletion is confirmed on disk.
*   In `PartialTableCache`, cleanup typically removes entries associated with flushed epochs entirely, as the goal is often to only cache *unflushed* changes or very recent reads.

This cleanup process runs periodically or is triggered by specific events, helping to manage memory usage.

## How to Use It: Enabling Cache for a Table

You usually don't interact with the `TableCache` object directly in your day-to-day code (`put`, `get`, etc.). Instead, you **configure** which cache strategy a `Table` should use when you define the database structure.

This configuration typically happens within your [DBDefinition / DBColumnFamilyDefinition](02_dbdefinition___dbcolumnfamilydefinition_.md) or potentially when using the [DBStoreBuilder](03_dbstorebuilder_.md). The `DBColumnFamilyDefinition` can specify the desired `CacheType` (`FULL_CACHE`, `PARTIAL_CACHE`, `NO_CACHE`).

**Conceptual Configuration (Inside DBDefinition):**

```java
import org.apache.hadoop.hdds.utils.db.DBColumnFamilyDefinition;
import org.apache.hadoop.hdds.utils.db.Codec;
import org.apache.hadoop.hdds.utils.db.StringCodec;
import org.apache.hadoop.hdds.utils.db.cache.TableCache.CacheType; // Import CacheType

// Assume StringCodec exists
Codec<String> stringCodec = StringCodec.get();

// Define "SessionData" table - small, frequently read, use Full Cache
DBColumnFamilyDefinition<String, String> sessionTableDefinition =
    new DBColumnFamilyDefinition<>(
        "SessionData",
        stringCodec,
        stringCodec,
        CacheType.FULL_CACHE // Specify Full Cache for this table!
    );

// Define "EventLog" table - large, many writes, use Partial Cache
DBColumnFamilyDefinition<Long, String> eventLogTableDefinition =
    new DBColumnFamilyDefinition<>(
        "EventLog",
        LongCodec.get(), // Assuming LongCodec exists
        stringCodec,
        CacheType.PARTIAL_CACHE // Specify Partial Cache!
    );

// Define "Archive" table - huge, rarely read, use No Cache
DBColumnFamilyDefinition<String, byte[]> archiveTableDefinition =
    new DBColumnFamilyDefinition<>(
        "Archive",
        stringCodec,
        ByteArrayCodec.get(), // Assuming ByteArrayCodec exists
        CacheType.NO_CACHE // Specify No Cache!
    );
```

*Explanation:*
*   When creating the `DBColumnFamilyDefinition` for each table, we now pass an additional argument: the desired `TableCache.CacheType`.
*   The `db` library uses this information during the `DBStoreBuilder.build()` process to create the appropriate `TableCache` instance (`FullTableCache`, `PartialTableCache`, or `TableNoCache`) for each table.

**How `Table.get()` Uses the Cache (Conceptual):**

Once the cache is configured, the `Table` object you get from `store.getTable(...)` will automatically use it. You don't need to change your `get` or `put` calls.

```java
// Assume 'sessionTable' was obtained from a store built with the definition above
// (using FULL_CACHE for SessionData)

try {
    String sessionId = "session-abc";
    System.out.println("Attempting to get session: " + sessionId);

    // Your code still just calls table.get()
    String sessionData = sessionTable.get(sessionId);

    // What happens internally (simplified):
    // 1. Table checks its configured TableCache (FullTableCache instance).
    // 2. Cache lookup for CacheKey("session-abc").
    // 3. CACHE HIT! Cache returns CacheValue containing "user=Alice,role=admin".
    // 4. Table extracts "user=Alice,role=admin" and returns it to your code.
    //    (Disk access was avoided!)

    if (sessionData != null) {
        System.out.println("Session data retrieved (likely from cache): " + sessionData);
    } else {
        System.out.println("Session data not found.");
        // If it was a CACHE MISS:
        // 4b. Table would then read from disk.
        // 5b. If found on disk, table might put it into the FullTableCache.
        // 6b. Table returns the data read from disk.
    }

} catch (IOException e) {
    System.err.println("Error getting session data: " + e.getMessage());
}
```

*Explanation:*
*   Your application code simply uses `table.get()`.
*   The magic happens inside the `Table` implementation. It consults its associated `TableCache` first.
*   If the data is in the cache (a hit), it's returned immediately, avoiding slow disk I/O.
*   If not (a miss), the table fetches from disk and might update the cache for next time.

### Understanding Cache Results (`lookup`)

Sometimes, especially when iterating or for more complex logic, you might want to know *if* something is in the cache without necessarily getting the value, or understand the cache's certainty. The `TableCache` interface provides a `lookup(CacheKey<KEY> key)` method returning a `CacheResult<VALUE>`.

The `CacheResult` contains:

*   `CacheStatus`: An enum indicating the result:
    *   `EXISTS`: The key was found in the cache, and the value is valid (not marked for deletion).
    *   `NOT_EXIST`: The key was found in the cache, but it's marked for deletion OR (in `FullTableCache`) the key was definitively not found. You can be sure it doesn't exist.
    *   `MAY_EXIST`: The key was *not* found in the `PartialTableCache`. Since the partial cache doesn't hold everything, the key *might* still exist on disk. You need to check the disk to be sure.
*   `CacheValue<VALUE>`: The actual cached value (or null if status is `NOT_EXIST` or `MAY_EXIST`).

```java
// Conceptual use of lookup
CacheKey<String> keyToLookup = new CacheKey<>("some-key");
CacheResult<String> result = eventLogTableCache.lookup(keyToLookup); // Assume eventLogTableCache is PartialCache

switch (result.getCacheStatus()) {
    case EXISTS:
        System.out.println("Found in cache: " + result.getValue().getCacheValue());
        break;
    case NOT_EXIST:
        System.out.println("Definitely does not exist (or marked deleted in cache).");
        break;
    case MAY_EXIST:
        System.out.println("Not in partial cache, might be on disk. Need to check DB.");
        // proceed to read from disk...
        break;
}
```

## Under the Hood: The Cache Lookup Flow

Let's trace what happens during a `table.get(key)` call when a cache is enabled.

1.  **`Table.get(key)` Called:** Your application calls `get` on the `TypedTable`.
2.  **Key Encoding:** The `TypedTable` encodes the Java key object into a `byte[]` key using its `KeyCodec`. It wraps this in a `CacheKey`.
3.  **Cache Lookup:** The `TypedTable` (or its internal helper) calls `cache.lookup(cacheKey)` on its configured `TableCache` instance.
4.  **Cache Logic:** The `TableCache` implementation (`FullTableCache` or `PartialTableCache`) checks its internal map (`ConcurrentSkipListMap` or `ConcurrentHashMap`) for the `cacheKey`.
    *   **Hit (Exists):** If found, and the `CacheValue` holds a non-null value, it returns `CacheResult(EXISTS, cacheValue)`.
    *   **Hit (Deleted):** If found, but the `CacheValue` holds a null value (marked for delete), it returns `CacheResult(NOT_EXIST, null)`.
    *   **Miss (Full Cache):** If not found in `FullTableCache`, it returns `CacheResult(NOT_EXIST, null)`.
    *   **Miss (Partial Cache):** If not found in `PartialTableCache`, it returns `CacheResult(MAY_EXIST, null)`.
5.  **Handle Cache Result:** The `TypedTable` receives the `CacheResult`.
    *   If `EXISTS`, it decodes the value from the `CacheValue` using its `ValueCodec` and returns the Java object. **(Fast path - done!)**
    *   If `NOT_EXIST`, it immediately returns `null`. **(Fast path - done!)**
    *   If `MAY_EXIST` (only from Partial Cache), it proceeds to the next step.
6.  **Disk Read (on Miss/MayExist):** The `TypedTable` asks the underlying `RDBTable` to read the raw `byte[]` key from the disk (RocksDB).
7.  **Disk Result:** The `RDBTable` returns the raw `byte[]` value from disk, or null if not found there either.
8.  **Value Decoding & Cache Update:**
    *   If data was found on disk, the `TypedTable` decodes the raw value bytes into a Java object using `ValueCodec`.
    *   **Crucially (for Partial Cache especially):** The `TypedTable` might now call `cache.put(cacheKey, newCacheValue)` to add the freshly read data into the cache, so subsequent reads might hit the cache.
    *   The decoded Java object is returned to the application.
    *   If data was not found on disk either, `null` is returned.

```mermaid
sequenceDiagram
    participant UC as User Code
    participant Table as TypedTable
    participant Cache as TableCache
    participant RDB as RDBTable (Disk)

    UC->>+Table: get("myKey")
    Table->>Table: Create CacheKey("myKey")
    Table->>+Cache: lookup(CacheKey("myKey"))
    alt Cache Hit (EXISTS)
        Cache-->>-Table: CacheResult(EXISTS, cacheValue)
        Table->>Table: Decode value from cacheValue
        Table-->>-UC: Decoded value (Fast!)
    else Cache Hit (NOT_EXIST)
        Cache-->>-Table: CacheResult(NOT_EXIST, null)
        Table-->>-UC: null (Fast!)
    else Cache Miss (MAY_EXIST or FullCache Miss)
        Cache-->>-Table: CacheResult(MAY_EXIST/NOT_EXIST, null)
        Table->>+RDB: get(keyBytes)
        RDB-->>-Table: valueBytes (or null)
        alt Found on Disk
            Table->>Table: Decode valueBytes
            Table->>+Cache: put(CacheKey, new CacheValue)
            Cache-->>-Table: OK
            Table-->>-UC: Decoded value
        else Not Found on Disk
             Table-->>-UC: null
        end
    end
```

*Explanation:* The diagram shows the `Table` checking the `Cache` first. If it gets a definitive answer (EXISTS/NOT_EXIST), it returns quickly. Only on a miss (especially MAY_EXIST from a partial cache) does it proceed to query the `RDBTable` (representing disk access), potentially updating the cache afterward.

**Code Glimpse:**

*   `TableCache.java`: The interface defining core cache operations.
    ```java
    // Simplified from TableCache.java
    public interface TableCache<KEY, VALUE> {
        // Lookup returning status and value
        CacheResult<VALUE> lookup(CacheKey<KEY> cachekey);

        // Get raw CacheValue (might be null)
        CacheValue<VALUE> get(CacheKey<KEY> cacheKey);

        // Add/update entry
        void put(CacheKey<KEY> cacheKey, CacheValue<VALUE> value);

        // Trigger cleanup based on completed epochs
        void cleanup(List<Long> epochs);

        // Load initial data (mainly for FullTableCache on startup)
        void loadInitial(CacheKey<KEY> cacheKey, CacheValue<VALUE> cacheValue);

        CacheType getCacheType();
        CacheStats getStats();
        int size();
        Iterator<Map.Entry<CacheKey<KEY>, CacheValue<VALUE>>> iterator();
    }
    ```
*   `FullTableCache.java`: Uses `ConcurrentSkipListMap` to store all entries. `lookup` returns `EXISTS` or `NOT_EXIST`.
    ```java
    // Simplified concept from FullTableCache.java
    public class FullTableCache<KEY, VALUE> implements TableCache<KEY, VALUE> {
        private final Map<CacheKey<KEY>, CacheValue<VALUE>> cache =
            new ConcurrentSkipListMap<>();
        private final NavigableMap<Long, Set<CacheKey<KEY>>> epochEntries =
            new ConcurrentSkipListMap<>(); // Tracks epochs mainly for deletes
        // ... lock, executorService ...

        @Override
        public CacheResult<VALUE> lookup(CacheKey<KEY> cachekey) {
            CacheValue<VALUE> cachevalue = cache.get(cachekey);
            // ... stats ...
            if (cachevalue == null) {
                // Not found in the full cache means it doesn't exist
                return new CacheResult<>(CacheStatus.NOT_EXIST, null);
            } else {
                if (cachevalue.getCacheValue() != null) {
                    return new CacheResult<>(CacheStatus.EXISTS, cachevalue);
                } else {
                    // Found but marked for delete
                    return new CacheResult<>(CacheStatus.NOT_EXIST, null);
                }
            }
        }
        // ... put adds to cache and epochEntries if value is null ...
        // ... cleanup removes deleted entries based on epoch ...
    }
    ```
*   `PartialTableCache.java`: Uses `ConcurrentHashMap` for fast lookups of cached subset. `lookup` can return `MAY_EXIST`.
    ```java
    // Simplified concept from PartialTableCache.java
    public class PartialTableCache<KEY, VALUE> implements TableCache<KEY, VALUE> {
        private final Map<CacheKey<KEY>, CacheValue<VALUE>> cache =
            new ConcurrentHashMap<>();
        private final NavigableMap<Long, Set<CacheKey<KEY>>> epochEntries =
            new ConcurrentSkipListMap<>(); // Tracks all entries by epoch
        // ... executorService ...

        @Override
        public CacheResult<VALUE> lookup(CacheKey<KEY> cachekey) {
            CacheValue<VALUE> cachevalue = cache.get(cachekey);
            // ... stats ...
            if (cachevalue == null) {
                // Not found in partial cache, it might still be on disk
                return (CacheResult<VALUE>) MAY_EXIST;
            } else {
                if (cachevalue.getCacheValue() != null) {
                    return new CacheResult<>(CacheStatus.EXISTS, cachevalue);
                } else {
                    // Found but marked for delete
                    return new CacheResult<>(CacheStatus.NOT_EXIST, null);
                }
            }
        }
        // ... put adds to cache and epochEntries ...
        // ... cleanup removes entries based on epoch ...
    }
    ```
*   `CacheKey.java` / `CacheValue.java`: Simple wrappers holding the key/value and epoch.

    ```java
    // Simplified CacheKey.java
    public class CacheKey<KEY> implements Comparable<CacheKey<KEY>> {
        private final KEY key;
        // ... constructor, getters, equals, hashCode, compareTo ...
    }

    // Simplified CacheValue.java
    public final class CacheValue<VALUE> {
        private final VALUE value; // Can be null if marked for delete
        private final long epoch;
        // ... static factory methods 'get', constructor, getters ...
    }
    ```

This structure allows different caching strategies to be plugged in, while the `Table` interacts with them through the common `TableCache` interface.

## Conclusion

You've learned about `TableCache`, a crucial component for improving database read performance!

*   It acts as an **in-memory cache** (like a desk tray) to store frequently accessed data from a `Table` (the filing cabinet).
*   It significantly speeds up reads by serving data from RAM (**cache hits**) instead of slower disk.
*   Key strategies include:
    *   **`FullTableCache`**: Caches everything (good for small, frequently read tables).
    *   **`PartialTableCache`**: Caches recent/active items (good for larger tables or write-heavy workloads).
    *   **`TableNoCache`**: Disables caching.
*   Caching is typically **configured** per table in the [DBDefinition / DBColumnFamilyDefinition](02_dbdefinition___dbcolumnfamilydefinition_.md) and works automatically behind the scenes when you use `Table` methods like `get`.
*   Mechanisms like **epoch-based cleanup** help manage cache size and consistency.

By choosing the right caching strategy for your tables, you can significantly enhance the responsiveness of your application. This concludes our core tutorial on the `db` library concepts!

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)