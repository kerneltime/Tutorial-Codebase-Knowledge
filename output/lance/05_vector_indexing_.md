# Chapter 5: Vector Indexing - Your Super-Fast Similarity Search Tool

Welcome to Chapter 5! In [Chapter 4: LanceScanner - Your Smart Data Retrieval Tool](04_lancescanner_.md), we learned how to use the `LanceScanner` to read data, apply filters, and even perform basic vector similarity searches using the `nearest` parameter. While `scanner.nearest()` works out of the box, for very large datasets (millions or even billions of items!), comparing your query vector to *every single* item can be slow.

Imagine you have a massive online store with millions of product images. When a user uploads an image to find similar products, you need results *fast*. You can't afford to compare the uploaded image's vector to every image vector in your database one by one. This is where **Vector Indexing** comes to the rescue!

## What Problem Does Vector Indexing Solve? Speeding Up Similarity Search!

**Vector Indexing** is a core feature in Lance designed to make similarity searches incredibly fast, especially for machine learning applications like finding similar images, documents, audio clips, or any other data represented as "vectors."

Think of it like this:
*   **Without an index:** You have a giant, disorganized pile of a million books. To find a book similar to one you have, you'd have to look at every single book in the pile. Exhausting!
*   **With a vector index:** You've pre-organized these million books into a sophisticated, multi-level filing system. When you want to find a similar book, this system helps you quickly narrow down your search to just a few relevant sections, instead of the whole pile.

Lance uses clever indexing strategies, like **IVF_PQ (Inverted File Index with Product Quantization)**, to build this "filing system."

### Key Concepts: IVF and PQ

Let's break down IVF_PQ:

1.  **IVF (Inverted File Index) - The Main Filing Cabinets:**
    *   Imagine your books (data items/vectors) are grouped into several large filing cabinet drawers based on their general topics (rough clusters). These drawers are called "partitions" or "clusters."
    *   When you have a new book (query vector) and want to find similar ones, IVF first helps you identify which one or few drawers (partitions) are most likely to contain similar books. You only need to search in these selected drawers. This drastically reduces the number of books you need to examine.

    ```mermaid
    graph TD
        A[Millions of Vectors] --> B{IVF Index};
        B -- Divides into --> C1[Partition 1 (e.g., Animal Images)];
        B -- Divides into --> C2[Partition 2 (e.g., Landscape Images)];
        B -- Divides into --> C3[Partition ...];
        B -- Divides into --> C_N[Partition N (e.g., Food Images)];

        Q[Query Vector (New Image)] --> B;
        B -- Directs to --> C2;
        subgraph Search Space
            C2
        end
    ```

2.  **PQ (Product Quantization) - Compressed Summaries in Each Drawer:**
    *   Now, inside each selected drawer (IVF partition), you still might have many books. Instead of comparing your query book to every full book in the drawer, PQ helps by creating highly compressed "summaries" (PQ codes) for each book.
    *   These summaries are much smaller and faster to compare. So, Lance first compares the summary of your query book to the summaries of all books in the chosen drawer(s). This quickly gives you a shortlist of the most promising candidates.
    *   **How PQ works (simplified):** It takes a large vector, splits it into several smaller sub-vectors, and then "quantizes" each sub-vector by finding its closest match from a small, pre-computed "codebook" of representative sub-vectors. The original vector is then represented by a short list of these codebook entry IDs.

    ```mermaid
    graph TD
        subgraph IVF Partition (e.g., Landscape Images)
            V1[Vector 1 (Full)] -- PQ Compression --> PQ1[PQ Code for V1 (Small)];
            V2[Vector 2 (Full)] -- PQ Compression --> PQ2[PQ Code for V2 (Small)];
            V_M[Vector M (Full)] -- PQ Compression --> PQM[PQ Code for V_M (Small)];
        end
        QV[Query Vector] -- PQ Compression --> Q_PQ[PQ Code for Query];
        Q_PQ -- Fast Comparison --> PQ1;
        Q_PQ -- Fast Comparison --> PQ2;
        Q_PQ -- Fast Comparison --> PQM;
    ```

**IVF_PQ Together:**
IVF first narrows down the search to a few partitions. Then, within those partitions, PQ allows for very fast, approximate distance calculations using the compressed codes. Finally, Lance might retrieve the original, full vectors for the top few candidates found via PQ and re-calculate their exact distances to give you the final, most accurate results. This multi-step process is much faster than a brute-force search.

## Creating Your First Vector Index

Let's see how to create an IVF_PQ index on our `my_products.lance` dataset. We'll use the `description_embedding` column.

**Important Note for this Chapter's Example:**
For Product Quantization (PQ) to be most effective and illustrative, it works best when vector dimensions are divisible by the number of sub-vectors (`num_sub_vectors`) used in PQ. Our `description_embedding` from previous chapters had a dimension of 3. To better demonstrate PQ, let's **assume for this chapter** that our `description_embedding` vectors are 4-dimensional, and we'll adjust our sample data accordingly.

First, let's set up our dataset. If `my_products_indexed.lance` (we'll use a new name to avoid conflicts) doesn't exist, we create it with 4D vectors.

```python
import lance
import pandas as pd
import pyarrow as pa
import numpy as np # For generating sample vectors

# Path for our new dataset for indexing
dataset_path_indexed = "my_products_indexed.lance"
DIMENSION = 4 # For better PQ illustration

# 1. Define schema with 4D vectors
product_schema_4d = pa.schema([
    pa.field("product_id", pa.int32()),
    pa.field("name", pa.string()),
    pa.field("price", pa.float32()),
    pa.field("description_embedding", pa.list_(pa.float32(), DIMENSION)) # Fixed-size list
])

# 2. Create sample data if dataset doesn't exist
try:
    dataset = lance.dataset(dataset_path_indexed)
    print(f"Opened existing dataset: {dataset.uri}")
except Exception:
    print(f"Dataset '{dataset_path_indexed}' not found. Creating it...")
    data_4d = [
        {"product_id": 1, "name": "Laptop X", "price": 1200.50, "description_embedding": np.random.rand(DIMENSION).astype(np.float32).tolist()},
        {"product_id": 2, "name": "Mouse Y", "price": 25.00, "description_embedding": np.random.rand(DIMENSION).astype(np.float32).tolist()},
        {"product_id": 3, "name": "Keyboard Z", "price": 75.75, "description_embedding": np.random.rand(DIMENSION).astype(np.float32).tolist()},
        # Add more data for a more realistic indexing scenario (e.g., 1000 rows)
        # For brevity, we'll stick to a few for this example.
        # In a real scenario, you'd want at least num_partitions * sample_rate (e.g., 2 * 32 = 64) rows.
        {"product_id": 4, "name": "Monitor A", "price": 300.00, "description_embedding": np.random.rand(DIMENSION).astype(np.float32).tolist()},
        {"product_id": 5, "name": "Webcam B", "price": 50.00, "description_embedding": np.random.rand(DIMENSION).astype(np.float32).tolist()},
    ]
    df_4d = pd.DataFrame(data_4d)
    dataset = lance.write_dataset(df_4d, dataset_path_indexed, schema=product_schema_4d)
    print(f"Created and wrote new dataset to: {dataset_path_indexed}")

# Our dataset object
print(f"Dataset version: {dataset.version}")
```

Now that we have a dataset, we can create an index using the `dataset.create_index()` method.

```python
# (Continuing from the previous script)

# Column to build the index on
vector_column_name = "description_embedding"

# Index parameters for IVF_PQ
# For a real dataset, you'd tune these. For our tiny dataset, these are illustrative.
num_partitions = 2  # How many "drawers" (IVF clusters)
num_sub_vectors = 2 # How many parts to split each vector into for PQ
                    # Dimension (4) must be divisible by num_sub_vectors (2)

try:
    print(f"\nAttempting to create an IVF_PQ index on '{vector_column_name}'...")
    dataset.create_index(
        vector_column_name,
        index_type="IVF_PQ",
        num_partitions=num_partitions,
        num_sub_vectors=num_sub_vectors,
        # metric_type="L2" # Default is L2, can also be "cosine" or "dot"
        # replace=True # If an index already exists, replace it. Default is False.
    )
    print(f"Successfully created or ensured index exists on '{vector_column_name}'.")
    print(f"New dataset version after indexing: {dataset.manifest.version}") # After indexing, the dataset version is updated.
    # Re-open the dataset to reflect the new version with the index
    dataset = lance.dataset(dataset_path_indexed)
    print(f"Dataset re-opened. Current version: {dataset.version}")

except Exception as e:
    print(f"Error creating index: {e}")
    print("Note: For very small datasets, some index parameters might need adjustment or more data.")

# Let's check the indices on the dataset
print("\nIndices in the dataset:")
for index_info in dataset.list_indices():
    print(f"  - Name: {index_info['name']}, Type: {index_info['type']}, Columns: {index_info['fields']}")
```

**Explanation:**
*   `dataset.create_index(vector_column_name, ...)`: This is the command to build an index.
    *   `vector_column_name`: The name of your vector column (e.g., `"description_embedding"`).
    *   `index_type="IVF_PQ"`: Specifies the type of index. Lance supports others too, but IVF_PQ is common.
    *   `num_partitions`: The number of IVF partitions (our "filing cabinet drawers"). A common starting point is `sqrt(number_of_rows)`. For our tiny 5-row dataset, 2 is just for illustration.
    *   `num_sub_vectors`: For PQ, how many sub-vectors to divide each original vector into. The dimension of the original vector (4 in our case) must be divisible by `num_sub_vectors` (2 in our case). So, each 4D vector will be split into 2 sub-vectors of 2D each. PQ will then quantize these 2D sub-vectors.
    *   `replace=True` (optional): If you try to create an index that already exists, Lance will throw an error unless `replace=True`.
*   Creating an index is an operation that modifies the dataset (by adding index files and updating metadata). This usually results in a new version of the dataset.
*   `dataset.list_indices()`: Shows you the indexes that have been built.

**Output (will vary slightly due to random data and timestamps):**
```
Opened existing dataset: my_products_indexed.lance
Dataset version: 1

Attempting to create an IVF_PQ index on 'description_embedding'...
Successfully created or ensured index exists on 'description_embedding'.
New dataset version after indexing: 2
Dataset re-opened. Current version: 2

Indices in the dataset:
  - Name: __lance_idx_14241857002612041717, Type: IVF_PQ, Columns: ['description_embedding']
```
*(The index name is auto-generated if not specified).*

### Searching with an Index

Now, when you perform a vector search using `dataset.scanner().nearest()`, Lance will automatically try to use any available compatible index on that column.

```python
# (Continuing from the previous script)

# A sample query vector (must be 4D)
query_vector = np.random.rand(DIMENSION).astype(np.float32).tolist()
k_neighbors = 2 # Number of nearest neighbors to find

print(f"\nPerforming a vector search for {k_neighbors} neighbors...")
scanner = dataset.scanner(
    nearest={
        "column": vector_column_name,
        "q": query_vector,
        "k": k_neighbors,
        # "nprobes": 1 # How many IVF partitions to search. Default depends on index.
                      # Higher nprobes = more accuracy, slower.
    }
)
results = scanner.to_pyarrow()

print("Search results (should now use the index if available):")
print(results.to_pandas())
```

**Expected Output (data will be random, but structure is key):**
```
Performing a vector search for 2 neighbors...
Search results (should now use the index if available):
   product_id      name    price  description_embedding  _distance
1           2   Mouse Y    25.00     [0.xx, 0.yy, ...]   0.123...
0           1  Laptop X  1200.50     [0.aa, 0.bb, ...]   0.456...
```
While the search syntax is the same as in [Chapter 4: LanceScanner - Your Smart Data Retrieval Tool](04_lancescanner_.md), Lance is now much smarter. It detects the IVF_PQ index on `description_embedding` and uses it to:
1.  Quickly find the `nprobes` most relevant IVF partitions for your `query_vector`.
2.  Within those partitions, use the PQ codes for fast approximate distance comparisons.
3.  Potentially re-rank the top candidates using full vector distances.

This makes the search significantly faster on large datasets compared to a brute-force (flat) search. For our tiny 5-row dataset, the speed difference won't be noticeable, but the principle is the same.

## How Indexing Works Under the Hood

### Building an IVF_PQ Index

When you call `dataset.create_index(...)`:

```mermaid
sequenceDiagram
    participant User
    participant Dataset as LanceDataset
    participant IndexBuilder as Internal Index Builder (Rust)
    participant Storage

    User->>Dataset: create_index("embedding_col", type="IVF_PQ", params...)
    Dataset->>IndexBuilder: Request to build IVF_PQ index
    IndexBuilder->>Dataset: Sample data for IVF training (subset of vectors)
    IndexBuilder->>IndexBuilder: **1. Train IVF Centroids:** Run k-means on sample to find 'num_partitions' cluster centers.
    IndexBuilder->>Dataset: Sample data for PQ training (residuals or full vectors)
    IndexBuilder->>IndexBuilder: **2. Train PQ Codebooks:** For each sub-vector, run k-means to create codebooks.
    IndexBuilder->>Dataset: Read all vectors from "embedding_col"
    IndexBuilder->>IndexBuilder: **3. Populate Index:** For each vector: <br/> a) Assign to nearest IVF partition. <br/> b) Compute its PQ code.
    IndexBuilder->>Storage: Write index files (IVF partition lists, PQ codes, centroids, codebooks)
    IndexBuilder->>Storage: Update dataset manifest with new index info & version.
    Storage-->>IndexBuilder: Confirmation
    IndexBuilder-->>Dataset: Index built successfully
    Dataset-->>User: New dataset version available
end
```

1.  **IVF Training (Centroid Calculation):**
    *   Lance takes a sample of your vectors from the specified column.
    *   It runs a clustering algorithm (like k-means) on this sample to find `num_partitions` representative vectors. These become the "centroids" (centers) of your IVF partitions. Each centroid defines a "drawer" in our filing cabinet.
2.  **PQ Training (Codebook Generation):**
    *   For PQ, Lance again samples vectors (often, it uses the "residual" vectors – the difference between a vector and its assigned IVF centroid).
    *   Each sampled vector is split into `num_sub_vectors`.
    *   For each set of sub-vectors at a given position (e.g., all the first sub-vectors, all the second sub-vectors, etc.), Lance runs k-means to create a "codebook" of 256 (typically) representative sub-vectors. This codebook is used to compress the sub-vectors.
3.  **Index Population:**
    *   Lance goes through all the vectors in your dataset.
    *   For each vector:
        *   It finds the closest IVF centroid and assigns the vector to that partition.
        *   It calculates the vector's PQ code by splitting it into sub-vectors and finding the closest entry in the corresponding PQ codebook for each sub-vector.
    *   This information (which partition each vector belongs to, and its PQ code) is stored efficiently in new index files within your Lance dataset directory.
4.  **Metadata Update:** The dataset's manifest file is updated to include information about the new index and a new version of the dataset is created.

### Using an IVF_PQ Index for Searching

When `dataset.scanner().nearest(...)` is called and an index exists:
```mermaid
sequenceDiagram
    participant User
    participant Scanner as LanceScanner
    participant IndexSearcher as Internal Index Searcher (Rust)
    participant Storage

    User->>Scanner: nearest(query_vec, k, nprobes)
    Scanner->>IndexSearcher: Pass query_vec, k, nprobes, and index info
    IndexSearcher->>Storage: Load IVF centroids
    IndexSearcher->>IndexSearcher: **1. Probe IVF Partitions:** Find 'nprobes' IVF partitions closest to query_vec.
    IndexSearcher->>Storage: For each probed partition: Load PQ codes and row IDs.
    IndexSearcher->>IndexSearcher: **2. PQ Search:** In probed partitions, compute approximate distances using PQ codes. <br/> Create a candidate list.
    IndexSearcher->>Storage: (Optional) Fetch full vectors for top N candidates.
    IndexSearcher->>IndexSearcher: **3. Re-ranking:** Compute exact distances for these candidates. <br/> Sort to get final 'k' results.
    IndexSearcher-->>Scanner: Top 'k' results (row IDs, distances)
    Scanner->>Storage: Fetch selected columns for the 'k' rows.
    Storage-->>Scanner: Data for 'k' rows.
    Scanner-->>User: Result Table (with data and distances)
end
```
1.  **IVF Probe:** Lance compares your `query_vector` to all the IVF partition centroids and selects the `nprobes` closest partitions. `nprobes` is a parameter you can tune: higher values mean more partitions are searched (potentially more accurate, but slower).
2.  **PQ Search in Probed Partitions:** Within these selected partitions, Lance uses the PQ codes. It calculates the PQ code for your `query_vector` and then efficiently computes *approximate* distances to all vectors (via their PQ codes) in the probed partitions. This quickly identifies a list of candidate similar items.
3.  **Re-ranking (Optional but common):** To improve accuracy, Lance often fetches the original, full vectors for a small number of top candidates from the PQ search. It then calculates the *exact* distances between your `query_vector` and these full vectors. The final `k` results are then selected based on these exact distances.

### Where Does This Happen in the Code?

*   **Python:**
    *   The main user-facing method is `dataset.create_index()` in `python/python/lance/dataset.py`. This method handles parsing your parameters and then calls into Lance's core Rust engine to do the heavy lifting.
    *   For more advanced, step-by-step index construction (e.g., training IVF and PQ models separately, then populating), Lance provides the `lance.indices.IndicesBuilder` class (in `python/python/lance/indices.py`). This is for users who need fine-grained control, perhaps for extremely large datasets or custom training routines. The functions used by `IndicesBuilder`, like `train_ivf_model` and `train_pq_model`, are exposed via `python/python/lance/lance/indices/__init__.pyi` and implemented in Rust.
    *   Helper functions for vector processing, including some used in GPU-accelerated index training, can be found in `python/python/lance/vector.py`.

*   **Java:**
    *   You would use the `dataset.createIndex(column, indexType, indexName, params, replace)` method found in `java/core/src/main/java/com/lancedb/lance/Dataset.java`.
    *   The parameters for the index (like IVF `numPartitions` or PQ `numSubVectors`) are often configured using specific builder classes, such as those related to `VectorIndexParams` in `java/core/src/main/java/com/lancedb/lance/index/vector/VectorIndexParams.java`. This file shows how different index types (IVF_FLAT, IVF_PQ, IVF_HNSW_PQ etc.) can be configured.

*   **Rust (The Core Engine):**
    *   The actual algorithms for k-means, product quantization, index building, and searching are implemented in Rust for maximum performance. The `lance-index` crate (as mentioned in `rust/lance-index/README.md`) contains much of this specialized logic.

The Python and Java libraries provide convenient wrappers around this powerful Rust core.

## Conclusion

You've now unlocked one of Lance's most powerful features: **Vector Indexing**! You've learned:
*   Why vector indexes are crucial for fast similarity searches in large datasets.
*   The basic concepts behind IVF_PQ: partitioning (IVF) and compression (PQ).
*   How to create an IVF_PQ index on a vector column in your `LanceDataset` using `dataset.create_index()`.
*   That indexed searches using `scanner.nearest()` are significantly more efficient.
*   A high-level overview of how index building and indexed search queries work under the hood.

Vector indexing transforms Lance from a simple data storage format into a high-performance vector database, capable of handling demanding AI/ML workloads.

Next, we'll delve deeper into the building blocks of a Lance dataset: the [LanceFragment & FragmentMetadata](06_lancefragment___fragmentmetadata_.md). Understanding fragments will give you more insight into how Lance organizes data physically on disk, which also contributes to its overall performance.

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)