# Chapter 7: Lance File Reader/Writer

Welcome to Chapter 7! In [Chapter 6: LanceFragment & FragmentMetadata](06_lancefragment___fragmentmetadata_.md), we learned how a Lance dataset is built from smaller pieces called fragments, each described by its metadata. These fragments, in turn, store their actual data in one or more physical files, typically ending with a `.lance` extension.

But what if you need to work directly with one of these individual `.lance` files? Perhaps you're debugging, building a custom data pipeline that outputs single Lance files, or you simply want to understand the lowest level of data storage in Lance. This is where `LanceFileReader` and `LanceFileWriter` come in.

## The "Single Document" Analogy

Think of a [LanceDataset](03_lancedataset_.md) as an entire filing cabinet, managing many documents (data files) and their versions. The `LanceFileReader` and `LanceFileWriter` are like having specific tools to open, read, or write a *single document file* from that cabinet, or even one that's not yet in any cabinet.

They provide lower-level access compared to the `LanceDataset` object, focusing on the content and structure of one physical `.lance` file.

**What problem do they solve?**
They allow you to:
*   Write data directly into a new, single `.lance` file.
*   Read data and metadata from an existing single `.lance` file.
*   Understand the raw structure of how Lance stores data columns.

Let's explore how to use these tools. For our examples, we'll primarily use Python, but we'll point out the corresponding components in Java.

## Writing a Single `.lance` File with `LanceFileWriter`

Imagine you have a small batch of data, maybe generated from a quick experiment, and you want to save it in the Lance format as a single file, without the overhead of creating a full dataset structure.

`LanceFileWriter` allows you to do just that. You'll need:
1.  A path where the file will be saved.
2.  A schema defining the structure of your data (refer to [Chapter 1: Schema - The Blueprint for Your Data](01_schema_.md)).
3.  The data itself, usually as an Apache Arrow `RecordBatch` or `Table`.

Let's create a simple `.lance` file.

```python
import pyarrow as pa
from lance.file import LanceFileWriter # Note the specific import

# 1. Define a simple schema
schema = pa.schema([
    pa.field("id", pa.int32()),
    pa.field("value", pa.string())
])

# 2. Create some sample data as an Arrow RecordBatch
data = [
    pa.array([1, 2, 3], type=pa.int32()),
    pa.array(["apple", "banana", "cherry"], type=pa.string())
]
record_batch = pa.RecordBatch.from_arrays(data, schema=schema)

# 3. Specify the path for our single .lance file
file_path = "my_single_data_file.lance"

# 4. Write the data using LanceFileWriter
# Using 'with' ensures the writer is properly closed
with LanceFileWriter(file_path, schema) as writer:
    writer.write_batch(record_batch)
    # You could write more batches here if needed
    # writer.write_batch(another_record_batch)

print(f"Successfully created single Lance file: {file_path}")
print(f"This file contains {record_batch.num_rows} rows.")
```

**What's happening here?**
1.  We define a PyArrow `schema`.
2.  We create a `record_batch` containing our data.
3.  We specify `file_path` for our output.
4.  `with LanceFileWriter(file_path, schema) as writer:`: This creates a `LanceFileWriter` object.
    *   `file_path`: Where the `.lance` file will be created.
    *   `schema`: The schema of the data we intend to write.
    *   The `with` statement ensures that `writer.close()` is called automatically when we're done, which finalizes the file.
5.  `writer.write_batch(record_batch)`: This takes our Arrow `RecordBatch` and writes its contents to the file in Lance's optimized columnar format.

**Output/Result:**
```
Successfully created single Lance file: my_single_data_file.lance
This file contains 3 rows.
```
A new file named `my_single_data_file.lance` is created in your current directory. This file contains our 3 rows of data, structured according to the schema we provided.

## Reading a Single `.lance` File with `LanceFileReader`

Now that we've created `my_single_data_file.lance`, let's say we want to inspect its contents or read the data back. `LanceFileReader` is the tool for this.

```python
import pyarrow as pa
from lance.file import LanceFileReader # Note the specific import
import pandas as pd # For nicer display

# 1. Path to the .lance file we created earlier
file_path = "my_single_data_file.lance"

# 2. Create a LanceFileReader
try:
    reader = LanceFileReader(file_path)
except Exception as e:
    print(f"Could not open file '{file_path}'. Make sure it was created by the previous script.")
    print(f"Error: {e}")
    exit()


# 3. Read file metadata
file_metadata = reader.metadata()
print(f"\n--- File Metadata for: {file_path} ---")
print(f"Number of rows: {file_metadata.num_rows}")
print("Schema stored in file:")
print(file_metadata.schema)

# 4. Read all data from the file
# .read_all() returns a ReaderResults object, then .to_table() converts to Arrow Table
arrow_table = reader.read_all().to_table()

print("\n--- Data read from file: ---")
print(arrow_table.to_pandas()) # Display as a Pandas DataFrame

# 5. (Optional) Read a specific range of rows
# For example, read 2 rows starting from row index 1
if file_metadata.num_rows >= 3:
    range_table = reader.read_range(start=1, num_rows=2).to_table()
    print("\n--- Reading rows 1-2 (0-indexed): ---")
    print(range_table.to_pandas())

```

**What's happening here?**
1.  We specify the `file_path` of the `.lance` file.
2.  `reader = LanceFileReader(file_path)`: This creates a `LanceFileReader` object, opening the specified file and reading its initial metadata.
3.  `file_metadata = reader.metadata()`: This method returns a `LanceFileMetadata` object, which contains information like the total number of rows and the schema stored within the file.
4.  `arrow_table = reader.read_all().to_table()`:
    *   `reader.read_all()`: Instructs the reader to fetch all rows from the file. This returns a utility object.
    *   `.to_table()`: Converts the read data into a PyArrow `Table`.
5.  `reader.read_range(start=1, num_rows=2)`: This is an example of more selective reading. It reads `num_rows` (2) starting from the row at `start` index (1). There's also `reader.take_rows([indices])` for reading specific, non-contiguous rows.

**Expected Output:**
```
--- File Metadata for: my_single_data_file.lance ---
Number of rows: 3
Schema stored in file:
id: int32
value: string

--- Data read from file: ---
   id   value
0   1   apple
1   2  banana
2   3  cherry

--- Reading rows 1-2 (0-indexed): ---
   id   value
0   2  banana
1   3  cherry
```
This confirms we can successfully read back both the metadata and the data from our single `.lance` file.

## Under the Hood: How Do They Work?

### `LanceFileWriter`

When you use `LanceFileWriter`:

```mermaid
sequenceDiagram
    participant User
    participant PyFileWriter as LanceFileWriter (Python)
    participant RustCoreWriter as Rust Core Writer
    participant Storage

    User->>PyFileWriter: LanceFileWriter("file.lance", schema)
    Note over PyFileWriter: Initializes RustCoreWriter

    User->>PyFileWriter: writer.write_batch(arrow_batch)
    PyFileWriter->>RustCoreWriter: Pass Arrow Batch
    RustCoreWriter->>RustCoreWriter: Encodes data (columnar, pages)
    RustCoreWriter->>Storage: Writes encoded data pages to "file.lance"

    User->>PyFileWriter: writer.close() (or end of 'with' block)
    PyFileWriter->>RustCoreWriter: Signal to finalize
    RustCoreWriter->>Storage: Writes file metadata (schema, page offsets, row count) to "file.lance"
    Storage-->>RustCoreWriter: Confirmation
    RustCoreWriter-->>PyFileWriter: Success
    PyFileWriter-->>User: File written
end
```

1.  **Initialization**: When `LanceFileWriter(path, schema)` is called, it prepares to write a new file. The schema is noted.
2.  **`write_batch(batch)`**:
    *   The Arrow `RecordBatch` is passed to Lance's core Rust engine.
    *   The data is converted from Arrow's in-memory format to Lance's on-disk columnar format. This involves encoding values, potentially compressing them, and organizing them into "pages" and "buffers" within the file.
    *   These encoded data chunks are written to the specified file path.
3.  **`close()` (or exiting a `with` block)**:
    *   This is a crucial step! The writer finalizes the file by writing out the file's metadata. This metadata includes:
        *   The complete schema.
        *   The total number of rows written.
        *   Information about where each column's data (pages/buffers) is located within the file (offsets and lengths).
        *   Version information about the Lance file format used.
    *   Without this metadata, the file would be incomplete and unreadable.

### `LanceFileReader`

When you use `LanceFileReader`:

```mermaid
sequenceDiagram
    participant User
    participant PyFileReader as LanceFileReader (Python)
    participant RustCoreReader as Rust Core Reader
    participant Storage

    User->>PyFileReader: LanceFileReader("file.lance")
    PyFileReader->>RustCoreReader: Initialize with file path
    RustCoreReader->>Storage: Reads file metadata (footer) from "file.lance"
    Storage-->>RustCoreReader: Raw metadata bytes
    RustCoreReader->>RustCoreReader: Parses metadata (schema, page map, row count)
    RustCoreReader-->>PyFileReader: Ready (metadata loaded)

    User->>PyFileReader: reader.metadata()
    PyFileReader-->>User: Returns parsed LanceFileMetadata

    User->>PyFileReader: reader.read_all().to_table()
    PyFileReader->>RustCoreReader: Request to read all data
    RustCoreReader->>Storage: Reads specific data pages/buffers based on parsed metadata
    Storage-->>RustCoreReader: Raw encoded data chunks
    RustCoreReader->>RustCoreReader: Decodes data into Arrow format
    RustCoreReader-->>PyFileReader: Arrow RecordBatch(es)
    PyFileReader-->>User: Arrow Table
end
```

1.  **Initialization**: `LanceFileReader(path)` opens the specified `.lance` file.
    *   It immediately reads the file metadata section (often located at the end of the file).
    *   This metadata is parsed to understand the file's schema, the total number of rows, and the layout of data for each column (where the pages/buffers are).
2.  **`metadata()`**: This method simply returns the `LanceFileMetadata` object that was parsed during initialization.
3.  **`read_all()` (or `read_range()`, `take_rows()`)**:
    *   The reader uses the parsed metadata to determine exactly which bytes to read from the file for the requested rows and columns.
    *   It seeks to those locations, reads the raw, encoded data.
    *   This raw data is then decoded from Lance's on-disk format back into Apache Arrow `RecordBatch`es.
    *   These batches are then combined (if `to_table()` is called) or provided as a stream.

This "metadata-first" approach for reading is efficient because Lance doesn't need to scan the entire file to understand its structure or to locate specific data.

## Code Pointers

Where do these components live in the `lance` codebase?

*   **Python:**
    *   The user-facing Python classes `LanceFileReader` and `LanceFileWriter` are defined in `python/python/lance/file.py`.
    *   These Python classes are thin wrappers around the core functionality implemented in Rust. The Python type hints in `python/python/lance/lance/__init__.pyi` show the underlying Rust-bound objects like `_LanceFileReader` and `_LanceFileWriter`.

*   **Java:**
    *   The Java equivalents are `com.lancedb.lance.file.LanceFileReader` (in `java/core/src/main/java/com/lancedb/lance/file/LanceFileReader.java`) and `com.lancedb.lance.file.LanceFileWriter` (in `java/core/src/main/java/com/lancedb/lance/file/LanceFileWriter.java`).
    *   Like their Python counterparts, these Java classes interface with the native Rust core to perform the actual file operations.

*   **Rust (Core Logic):**
    *   The fundamental logic for reading and writing the Lance file format resides in the `lance-file` Rust crate. You can see its mention in `rust/lance-file/README.md`. This crate handles the encoding/decoding, page management, and metadata serialization.

## Conclusion

You've now learned about `LanceFileReader` and `LanceFileWriter`, the tools for direct, low-level interaction with individual `.lance` data files.
*   `LanceFileWriter` allows you to create new, single `.lance` files from Arrow data.
*   `LanceFileReader` enables you to read metadata and data from existing `.lance` files.
*   These are essential for understanding the physical storage of data in Lance and for scenarios requiring fine-grained control over individual files.

While most day-to-day interactions with Lance will likely be through the higher-level [LanceDataset](03_lancedataset_.md) and [LanceScanner](04_lancescanner_.md) APIs, knowing about these file-level tools gives you a complete picture of how Lance works, from the "filing cabinet" down to the "individual documents."

This concludes our current series on the core concepts of Lance. We hope these chapters have provided you with a solid foundation to start building amazing things with Lance! Keep exploring the documentation and examples to discover even more capabilities.

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)