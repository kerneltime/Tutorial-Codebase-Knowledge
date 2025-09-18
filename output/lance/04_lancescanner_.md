# Chapter 4: LanceScanner - Your Smart Data Retrieval Tool

Welcome to Chapter 4! In [Chapter 3: LanceDataset - Your Data's Front Door](03_lancedataset_.md), we learned how to open and inspect our Lance datasets using the `LanceDataset` object. We now have our "data catalog" open and can see what's inside. But how do we actually read the data efficiently, especially if we only need specific parts of it?

That's where the **`LanceScanner`** comes in. It's your primary tool for reading data *from* a `LanceDataset` in a flexible and powerful way.

## What is a `LanceScanner`? Your Personal Data Assistant

Imagine you're at that special library (your `LanceDataset`) from Chapter 3. You don't just want to grab all the books off the shelves. Instead, you have specific needs:
*   "I only want the books by 'Author X'."
*   "And from those books, I only need Chapter 2."
*   "Oh, and can you give them to me five at a time?"
*   "Could you also find books most similar to this sample paragraph I have?"

A **`LanceScanner`** is like a highly configurable "search assistant" or a "query builder" for your dataset. You tell it:
*   **Which columns** you want (e.g., just the `name` and `price`, not the `description_embedding`).
*   **Any filters** to apply (e.g., only products where `price > 50`).
*   **How many rows** to get at once (batch size).
*   **How many rows in total** (`limit`).
*   Even advanced requests, like finding the **nearest neighbors** to a given vector (which we'll touch on in [Vector Indexing](05_vector_indexing_.md)).

The `LanceScanner` then figures out the most efficient way to retrieve exactly the data you asked for from the underlying dataset files (fragments). This is crucial for performance, especially with large datasets.

## Getting Started: Creating a `LanceScanner`

You don't create a `LanceScanner` from scratch. Instead, you ask your `LanceDataset` object to give you one. This is done using the `dataset.scanner()` method.

Let's set up our environment and get a `LanceDataset` object, just like in the previous chapter. (Ensure you have `my_products.lance` created from Chapter 2's code).

```python
import lance
import pandas as pd # For displaying data nicely
import pyarrow as pa # For data types

# Path to the dataset we created in Chapter 2
dataset_path = "my_products.lance"

# 1. Open the LanceDataset
try:
    dataset = lance.dataset(dataset_path)
    print(f"Successfully opened dataset: {dataset.uri}")
except Exception as e:
    print(f"Could not open dataset at '{dataset_path}'.")
    print(f"Please ensure you've run the code from Chapter 2 to create it.")
    print(f"Error: {e}")
    exit() # Can't proceed without the dataset

# 2. Get a basic scanner
# By default, it's configured to read all columns and all rows
scanner = dataset.scanner()
print(f"\nCreated a LanceScanner: {type(scanner)}")
```

**Explanation:**
1.  We open our `my_products.lance` dataset.
2.  `scanner = dataset.scanner()`: This call creates a `LanceScanner` instance associated with our `dataset`. At this point, no data has been read yet. We've just prepared our "search assistant".

**Output:**
```
Successfully opened dataset: my_products.lance

Created a LanceScanner: <class 'lance.dataset.LanceScanner'>
```

### Reading Data with the Scanner

Once you have a `LanceScanner`, you can tell it to fetch the data. A common way is to convert the results into an Apache Arrow Table using `scanner.to_pyarrow()`, or to iterate over batches of data using `scanner.to_batches()`.

```python
# (Continuing from the previous script)

# 3. Read all data using the basic scanner
print("\nReading all data with the default scanner:")
all_data_table = scanner.to_pyarrow()
print(all_data_table.to_pandas()) # Display as a Pandas DataFrame
```

**Expected Output (from Chapter 2's data):**
```
Reading all data with the default scanner:
   product_id      name    price  description_embedding
0           1    Laptop  1200.50      [0.1, 0.2, 0.3]
1           2     Mouse    25.00      [0.4, 0.5, 0.6]
2           3  Keyboard    75.75      [0.7, 0.8, 0.9]
```
This simple scanner, with no extra configuration, fetched all columns and all rows from our dataset.

## Customizing Your Data Request

The real power of `LanceScanner` comes from its configuration options. Let's tell our "assistant" more precisely what we need.

### 1. Selecting Specific Columns (`columns` parameter)

Often, you don't need every single column from your dataset. Reading fewer columns means less data to process, which is faster.

```python
# (Continuing from the previous script)

# Request only 'name' and 'price' columns
scanner_specific_columns = dataset.scanner(columns=["name", "price"])

print("\nReading only 'name' and 'price' columns:")
name_price_table = scanner_specific_columns.to_pyarrow()
print(name_price_table.to_pandas())
```

**Expected Output:**
```
Reading only 'name' and 'price' columns:
       name    price
0    Laptop  1200.50
1     Mouse    25.00
2  Keyboard    75.75
```
See? The `product_id` and `description_embedding` columns are gone, as requested!

### 2. Filtering Rows (`filter` parameter)

You can apply conditions to select only rows that match certain criteria, similar to a `WHERE` clause in SQL. The filter is a string expression.

```python
# (Continuing from the previous script)

# Request products where price is greater than 100
scanner_filtered = dataset.scanner(
    columns=["name", "price"],
    filter="price > 100" # SQL-like filter expression
)

print("\nReading products with price > 100:")
expensive_products_table = scanner_filtered.to_pyarrow()
print(expensive_products_table.to_pandas())
```

**Expected Output:**
```
Reading products with price > 100:
     name    price
0  Laptop  1200.50
```
Only the "Laptop" remains, as it's the only product with a price over 100.

### 3. Controlling How Much Data: `limit` and `batch_size`

*   `limit`: Specifies the maximum number of rows to return.
*   `batch_size`: When reading data, Lance can deliver it in chunks (batches). This is useful for processing large datasets without loading everything into memory at once.

Let's get the first 2 products, delivered in batches of 1. We'll use `scanner.to_batches()` which returns an iterator.

```python
# (Continuing from the previous script)

scanner_batched = dataset.scanner(
    limit=2,          # Get at most 2 rows
    batch_size=1      # Return them one row at a time
)

print("\nReading first 2 products, in batches of 1:")
batch_count = 0
for batch in scanner_batched.to_batches():
    batch_count += 1
    print(f"--- Batch {batch_count} ---")
    print(batch.to_pandas())
```

**Expected Output:**
```
Reading first 2 products, in batches of 1:
--- Batch 1 ---
   product_id    name   price  description_embedding
0           1  Laptop  1200.5      [0.1, 0.2, 0.3]
--- Batch 2 ---
   product_id   name  price  description_embedding
0           2  Mouse   25.0      [0.4, 0.5, 0.6]
```
We got two batches, each containing one row, and only the first two products overall.

### 4. A Glimpse into Vector Search (`nearest` parameter)

One of Lance's superpowers is fast vector search. If you have vector embeddings in your dataset (like our `description_embedding`), you can use the `LanceScanner` to find items whose embeddings are "closest" to a query vector.

```python
# (Continuing from the previous script)

# This is a conceptual example.
# Effective vector search usually requires a vector index,
# which we'll cover in Chapter 5: Vector Indexing.

# Imagine we want to find the 2 products most similar to a query vector
query_vector = [0.11, 0.22, 0.33] # Our example query vector

scanner_vector_search = dataset.scanner(
    nearest={
        "column": "description_embedding", # The vector column to search
        "q": query_vector,                 # The query vector
        "k": 2                             # Number of nearest neighbors to find
    }
)
print("\nConceptually performing a vector search (results depend on actual data and distance):")
try:
    vector_search_results = scanner_vector_search.to_pyarrow()
    print(vector_search_results.to_pandas())
    # The output will also include a '_distance' column
except Exception as e:
    print(f"Note: Vector search might require an index or specific setup. Error: {e}")
    print("We'll cover vector indexing in detail in the next chapter!")

```
**Conceptual Output (actual values will differ):**
```
Conceptually performing a vector search (results depend on actual data and distance):
   product_id    name    price  description_embedding  _distance
0           1  Laptop  1200.50      [0.1, 0.2, 0.3]   0.001...
1           2   Mouse    25.00      [0.4, 0.5, 0.6]   0.25...
```
The `nearest` parameter tells the scanner to perform a vector similarity search. It would return the `k` most similar items, along with their distances to the query vector. We'll dive deep into how this works and how to make it super-fast in [Chapter 5: Vector Indexing](05_vector_indexing_.md).

## How `LanceScanner` Works Its Magic (Under the Hood)

When you configure a `LanceScanner` and ask for data (e.g., by calling `to_pyarrow()` or `to_batches()`), Lance doesn't just blindly read everything. It's much smarter:

```mermaid
sequenceDiagram
    participant User
    participant Dataset as LanceDataset
    participant Scanner as LanceScanner
    participant QueryPlanner as Internal Query Planner
    participant FragmentReader as Fragment Reader
    participant Storage

    User->>Dataset: dataset.scanner(params: cols, filter, limit)
    Dataset-->>User: Returns Scanner instance (stores params)

    User->>Scanner: scanner.to_pyarrow()
    Scanner->>QueryPlanner: Query parameters (cols, filter, etc.)
    Note over QueryPlanner: Analyzes query, identifies relevant fragments & data ranges
    QueryPlanner-->>Scanner: Optimized access plan

    Scanner->>FragmentReader: Request data based on plan
    FragmentReader->>Storage: Reads only necessary data (specific columns/rows) from .lance files
    Storage-->>FragmentReader: Raw data chunks

    FragmentReader-->>Scanner: Processes chunks (applies filters, creates Arrow batches)
    Scanner-->>User: Arrow Table / RecordBatchReader
end
```

1.  **Query Plan Creation:** When you initiate the data read, the `LanceScanner` (often with help from Lance's core engine) creates an optimized "query plan." This plan figures out:
    *   Which data files ([LanceFragment & FragmentMetadata](06_lancefragment___fragmentmetadata_.md)) actually need to be accessed. If your filter is on an indexed column, or if metadata can prove a fragment doesn't contain relevant data, Lance might skip reading that fragment entirely!
    *   Which columns need to be read from those files.
    *   The most efficient order to perform operations.
2.  **Optimized Data Reading:** Lance reads only the necessary bytes from disk. Because Lance uses a columnar format, it can read just the `price` and `name` columns without touching the `description_embedding` data on disk if those are the only columns you requested.
3.  **Filtering and Processing:** Data is then filtered (if a `filter` was provided) and assembled into Arrow `RecordBatch`es according to your `batch_size`.
4.  **Vector Search Integration:** If a `nearest` neighbor query is involved, the scanner coordinates with the vector index (see [Vector Indexing](05_vector_indexing_.md)) to quickly find candidate rows, then fetches their full data if needed.

This intelligent processing is what makes `LanceScanner` efficient.

**Where does this happen in the code?**
*   **Python:** When you call `dataset.scanner(...)` (from `python/python/lance/dataset.py`), a `LanceScanner` object is created. This object holds your query parameters. The actual work of planning and execution is largely delegated to Lance's underlying Rust core, accessed via an internal `_Scanner` object (referenced in `python/python/lance/lance/__init__.pyi`).
    ```python
    # In python/python/lance/dataset.py (LanceDataset class)
    # def scanner(self, ...) -> LanceScanner:
    #     # ... creates a native _Scanner object from Rust core ...
    #     native_scanner_obj = self._dataset.scanner(...) # Call to Rust
    #     return LanceScanner(native_scanner_obj, self, ...)

    # In python/python/lance/dataset.py (LanceScanner class)
    # def to_pyarrow(self) -> pa.Table:
    #     return self._scanner.to_pyarrow() # _scanner is the native object
    ```
*   **Java:** In Java, you typically use `dataset.newScan(scanOptions)` (from `java/core/src/main/java/com/lancedb/lance/Dataset.java`). This returns a `com.lancedb.lance.ipc.LanceScanner` instance. The `ScanOptions` class (in `java/core/src/main/java/com/lancedb/lance/ipc/ScanOptions.java`) is used to specify query parameters. The `LanceScanner` then interfaces with the native Rust core to perform the scan.
    ```java
    // Conceptual Java usage
    // ScanOptions options = new ScanOptions.Builder()
    //     .columns(List.of("name", "price"))
    //     .filter("price > 100")
    //     .build();
    // LanceScanner scanner = dataset.newScan(options);
    // ArrowReader reader = scanner.scanBatches();
    // // ... process data from reader ...
    // reader.close();
    // scanner.close();
    ```

The key takeaway is that `LanceScanner` isn't just a simple loop; it's a sophisticated system for retrieving precisely the data you need, as efficiently as possible.

## Conclusion

You've now mastered the `LanceScanner`, your versatile tool for reading data from Lance datasets! You learned how to:
*   Create a scanner from a `LanceDataset`.
*   Select specific columns.
*   Filter rows based on conditions.
*   Control the amount of data using `limit` and process it in `batch_size` chunks.
*   Get a conceptual understanding of how it can be used for vector searches.
*   Appreciate the "under the hood" optimizations that make `LanceScanner` efficient.

The `LanceScanner` is fundamental to nearly all read operations in Lance. As we move forward, you'll see it used in many contexts.

Speaking of vector searches, how does Lance make them so fast, especially on massive datasets? The answer lies in **Vector Indexing**. Get ready to explore this powerful feature in our next chapter: [Vector Indexing](05_vector_indexing_.md)!

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)