# Chapter 2: Data Ingestion / Write Operations

Welcome back! In [Chapter 1: Schema - The Blueprint for Your Data](01_schema_.md), we learned how to define the structure of our data using schemas. Now, with a blueprint in hand, the next logical step is to actually *build* our dataset. How do we get our existing data into this new, efficient Lance format? That's what this chapter is all about: **Data Ingestion**, or more simply, **Write Operations**.

## What's the Goal? From Your Data to a Lance Dataset

Imagine you have a spreadsheet, a CSV file, or maybe a collection of product information stored in your Python program. This is your raw material. Data ingestion is the process of taking this raw material and transforming it into a structured, optimized Lance dataset on your computer's storage (or in the cloud).

Let's continue with our online store example from Chapter 1. We have product information like:
*   Product ID
*   Name
*   Price
*   Description Embedding (a list of numbers representing the description)

Our goal is to take this information, perhaps currently in a popular Python data structure like a Pandas DataFrame, and save it as a `.lance` dataset file (or directory).

## The Easy Way: `lance.write_dataset()`

For many common situations, especially when you have all your data ready in memory (like in a Pandas DataFrame or an Apache Arrow Table), Lance provides a very convenient, high-level function in Python: `lance.write_dataset()`.

Think of it like this:
*   You have a box full of organized documents (your DataFrame or Arrow Table).
*   `lance.write_dataset()` is like a super-efficient archiving service. You hand over the box and the blueprint (schema), and it quickly stores everything neatly into a special archive (the Lance dataset) for you.

Let's see how to use it. First, we need some data and the schema we defined in Chapter 1.

```python
import pyarrow as pa
import pandas as pd
import lance

# 1. Recall our schema from Chapter 1
product_schema = pa.schema([
    pa.field("product_id", pa.int32()),
    pa.field("name", pa.string()),
    pa.field("price", pa.float32()),
    # For simplicity in this example, let's make the embedding a fixed-size list
    # PyArrow fixed_size_list_type requires the size at schema definition.
    # If your embeddings have variable lengths, you'd use pa.list_(pa.float32())
    # and ensure all lists in your data are of the same length IF a fixed-size
    # vector index is to be built on it later. For now, pa.list_ is fine.
    pa.field("description_embedding", pa.list_(pa.float32()))
])

# 2. Let's create some sample data as a Pandas DataFrame
data = [
    {"product_id": 1, "name": "Laptop", "price": 1200.50, "description_embedding": [0.1, 0.2, 0.3]},
    {"product_id": 2, "name": "Mouse", "price": 25.00, "description_embedding": [0.4, 0.5, 0.6]},
    {"product_id": 3, "name": "Keyboard", "price": 75.75, "description_embedding": [0.7, 0.8, 0.9]},
]
df = pd.DataFrame(data)

# 3. Define where to save our Lance dataset
dataset_path = "my_products.lance"

# 4. Write the data!
lance.write_dataset(df, dataset_path, schema=product_schema)

print(f"Lance dataset created at: {dataset_path}")
```

**What's happening here?**
1.  We import necessary libraries: `pyarrow`, `pandas`, and `lance`.
2.  We define our `product_schema` (just like in Chapter 1). This tells Lance what our data looks like.
3.  We create some sample product data using a Pandas DataFrame. This is a very common way to hold data in Python.
4.  We choose a `dataset_path`. This is where our Lance dataset will be saved. Lance datasets are actually directories.
5.  The magic line: `lance.write_dataset(df, dataset_path, schema=product_schema)`.
    *   `df`: Our data (the Pandas DataFrame). You can also pass a PyArrow Table directly.
    *   `dataset_path`: Where to save it.
    *   `schema=product_schema`: Our blueprint. Lance will use this to understand and validate the data from the DataFrame. If you provide a PyArrow Table, the schema is often inferred, but explicitly providing it is good practice.

**Output/Result:**
After running this code, you won't see data printed to the console from the `write_dataset` call itself. Instead, a new directory named `my_products.lance` will be created in the same location as your script. This directory *is* your Lance dataset, containing various files in Lance's optimized format.

```
Lance dataset created at: my_products.lance
```

You've successfully ingested your data into Lance!

## Under the Hood: What `lance.write_dataset()` Does

When you call `lance.write_dataset()`, several things happen behind the scenes:

```mermaid
sequenceDiagram
    participant User
    participant write_dataset_func as lance.write_dataset()
    participant DataConverter as Data Converter
    participant LanceWriter as Lance Internal Writer
    participant Storage

    User->>write_dataset_func: Provides DataFrame, Path, Schema
    Note over write_dataset_func: If data is Pandas, convert to Arrow
    write_dataset_func->>DataConverter: DataFrame data
    DataConverter-->>write_dataset_func: Arrow Table data
    write_dataset_func->>LanceWriter: Pass Arrow Table & Schema
    Note over LanceWriter: Divides data into "fragments"
    LanceWriter->>Storage: Writes data files (data for each fragment)
    LanceWriter->>Storage: Writes metadata files (schema, version info, fragment list)
    Storage-->>LanceWriter: Confirmation
    LanceWriter-->>write_dataset_func: Success
    write_dataset_func-->>User: Dataset directory created
end
```

1.  **Data Conversion (if needed):** If you provide data like a Pandas DataFrame, Lance (often using PyArrow) first converts it into an Apache Arrow Table. Arrow is the format Lance uses internally.
2.  **Schema Validation:** Lance checks your data against the provided schema to make sure everything matches up (e.g., 'price' is indeed numbers, 'name' is text).
3.  **Data Fragmentation:** Lance doesn't usually write all your data into one enormous file. Instead, it intelligently breaks your data down into smaller, manageable pieces called **data fragments**. Each fragment might contain a subset of your rows. This is key for performance and scalability. We'll explore this more in [LanceFragment & FragmentMetadata](06_lancefragment___fragmentmetadata_.md).
4.  **Writing Data Files:** For each fragment, Lance writes the actual column data to one or more files in its specialized, optimized format.
5.  **Writing Metadata:** Lance also writes metadata files. These files describe the dataset:
    *   The schema.
    *   The list of data fragments and where to find their data files.
    *   Version information (Lance supports dataset versioning!).
    *   Other statistics and information to help read data quickly.

All these files are organized within the dataset directory (e.g., `my_products.lance/`).

## Finer-Grained Control: Beyond `write_dataset()`

`lance.write_dataset()` is fantastic for batch ingestion—when you have all your data ready. But what if:
*   You're dealing with a **continuous stream of data**?
*   You want to build up your dataset **incrementally, piece by piece** (fragment by fragment)?
*   You need very specific control over how each **fragment** is written?

For these more advanced scenarios, Lance offers lower-level tools. This is like choosing between quickly archiving that whole box of files (`write_dataset`) versus carefully filing each document individually into specific folders.

*   **In Python:** You might use `lance.LanceFileWriter` to write individual data files (which become part of fragments) or functions like `lance.write_fragments` to create fragments from streams of data. These give you more control over how data is physically laid out. We'll touch upon `LanceFileWriter` in more detail in [Lance File Reader/Writer](07_lance_file_reader_writer_.md).
*   **In Java:** You'd use methods like `Dataset.create()` (which can take an `ArrowArrayStream` for data input) along with `WriteParams` to specify how the dataset should be created. For even more granular control, Java also has ways to work with fragments more directly, perhaps involving `Fragment.create()`.

Let's look at a conceptual Java example for creating a dataset. The `Dataset.create()` method can be used with an `ArrowReader` (which provides data in Arrow format) and `WriteParams` to configure the writing process.

```java
// Conceptual Java Snippet
// (Full setup requires more boilerplate like imports and error handling)

// Assume:
// - 'allocator' is a pre-configured org.apache.arrow.memory.BufferAllocator
// - 'schema' is your org.apache.arrow.vector.types.pojo.Schema
// - 'dataReader' is an org.apache.arrow.vector.ipc.ArrowReader providing your data

import org.apache.arrow.memory.BufferAllocator;
import org.apache.arrow.memory.RootAllocator;
import org.apache.arrow.vector.ipc.ArrowReader; // For the concept
import org.apache.arrow.vector.types.pojo.Schema; // For the concept
import com.lancedb.lance.Dataset;
import com.lancedb.lance.WriteParams;
import com.lancedb.lance.fragment.Fragment; // For Fragment.create concept

// --- Conceptual Setup (replace with actual data loading) ---
BufferAllocator allocator = new RootAllocator();
// Schema schema = ... your schema defined as in Chapter 1 ...
// ArrowReader dataReader = ... your data source ...
// --- End Conceptual Setup ---

String datasetUri = "my_java_products.lance";

// Define Write Parameters
WriteParams.Builder paramsBuilder = new WriteParams.Builder();
paramsBuilder.withMode(WriteParams.WriteMode.CREATE); // Create a new dataset
// Optional: control how data is chunked into files/groups within fragments
// paramsBuilder.withMaxRowsPerFile(1024 * 1024);
// paramsBuilder.withMaxRowsPerGroup(1024);
WriteParams writeParams = paramsBuilder.build();

// High-level way to write a new dataset from an Arrow stream
// Dataset.create(allocator, dataReader, datasetUri, writeParams);

// For more granular control, you might create fragments individually
// (Conceptual - Fragment.create is a Python API, Java equivalent involves working with FragmentMetadata and commit operations)
// FragmentMetadata newFragment = Fragment.create(datasetUri, dataReader /* or specific batch */, ...);
// Then commit this fragment to a dataset.

System.out.println("Java dataset creation process would create: " + datasetUri);

// --- Conceptual Cleanup ---
// dataReader.close();
// allocator.close();
// --- End Conceptual Cleanup ---
```
This Java snippet illustrates that `Dataset.create()` is a common entry point. The `WriteParams` object allows you to specify things like:
*   `mode`: `CREATE` (make a new dataset), `APPEND` (add to existing), or `OVERWRITE` (replace existing).
*   `maxRowsPerFile`: How many rows to put in each underlying file within a fragment.
*   `maxRowsPerGroup`: How many rows to put in each row group within a file (for finer-grained access).

For now, as beginners, `lance.write_dataset()` in Python will be your primary tool. Understanding that more advanced options exist is good for future reference.

## Conclusion

You've now learned the fundamental ways to get your data *into* Lance!
*   For most cases, `lance.write_dataset()` in Python is the simplest and most convenient method to convert data from formats like Pandas DataFrames or Arrow Tables into a new Lance dataset.
*   Under the hood, Lance organizes data into fragments and writes optimized data and metadata files.
*   For more complex scenarios like streaming data or fine-grained fragment construction, Lance provides lower-level APIs like `LanceFileWriter` (Python) and `Dataset.create()` with `Fragment` operations (Java).

With your data now safely and efficiently stored in a Lance dataset, how do you actually *use* it? That's what we'll cover in the next chapter, where we introduce the [LanceDataset](03_lancedataset_.md) object, your gateway to reading and querying your data.

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)