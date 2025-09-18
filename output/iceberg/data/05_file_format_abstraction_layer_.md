# Chapter 5: File Format Abstraction Layer

In [Chapter 4: Delete Filter & Loader](04_delete_filter___loader_.md), we saw how the `DeleteLoader` needs to read information from delete files, which, just like data files, can be in different formats like Parquet, Avro, or ORC. This brings up a common challenge: how does Iceberg work with various file types without making your application code complicated?

The answer lies in Iceberg's "File Format Abstraction Layer." Let's explore what that means!

## The "Many Languages" Problem: Why We Need an Adapter

Imagine you're building a system that needs to store and retrieve documents. Some documents are in English (like Parquet files), some in Spanish (Avro), and others in French (ORC). Each "language" has its own grammar and dictionary (its own way of storing data and its own software libraries to read/write it).

If you had to teach your application to speak English, Spanish, *and* French fluently, your code would become very complex. You'd need separate logic for each language:
*   `if (document is English) { use English_Reader; }`
*   `else if (document is Spanish) { use Spanish_Reader; }`
*   `else if (document is French) { use French_Reader; }`

This is messy and hard to maintain. What if a new language, German (another file format), comes along? You'd have to update your code again!

**Our Use Case:** Let's say we want to write some employee records (as generic `Record` objects from [Chapter 1: Generic Row Representation (`Record` & `InternalRecordWrapper`)](01_generic_row_representation___record_____internalrecordwrapper___.md)) into an Iceberg table.
*   Initially, our table is configured to store data files in **Parquet** format.
*   Later, for performance reasons, an administrator changes the table's default file format to **ORC**.

Ideally, our application code for writing these employee records shouldn't need to change at all, even though the underlying file format has changed. How can Iceberg achieve this?

## The Universal Adapter: Iceberg's Abstraction Layer

Iceberg's File Format Abstraction Layer acts like a **universal adapter** or a **multilingual translator**.

*   **You (the application developer):** You speak a common language when you want to read or write data. You interact with generic interfaces. For example, when writing, you might use a `FileWriterFactory<Record>` and the `DataWriter<Record>` it produces (as seen in [Chapter 3: Generic Data Writing](03_generic_data_writing_.md)). When reading, you get `Record` objects from something like `IcebergGenerics.read(table)` (as seen in [Chapter 2: Generic Data Reading](02_generic_data_reading_.md)).
*   **Iceberg (The Universal Adapter):** Behind the scenes, Iceberg looks at the specific "language" (file format) required for a task.
    *   Is the table configured to write Parquet files? Iceberg picks its specialized "Parquet tool."
    *   Are you trying to read an Avro file? Iceberg picks its "Avro tool."
*   **Specialized Tools:** Iceberg has built-in support for common formats like Parquet, Avro, and ORC. These are like its expert translators for each language, living in modules like `iceberg-parquet`, `iceberg-avro`, and `iceberg-orc`.

This keeps your application code clean and **format-agnostic**. You tell Iceberg *what* you want to do (read/write data), and Iceberg figures out *how* to do it for the specific file format.

## How It Works in Practice

Let's see how this universal adapter helps with our use case of writing employee records.

**1. Writing Data (Example Revisited from Chapter 3)**

Remember how we used `GenericFileWriterFactory` to get a `DataWriter` in [Chapter 3: Generic Data Writing](03_generic_data_writing_.md)?

```java
// Assume 'employeesTable' is an existing Iceberg Table object
// Table employeesTable = ... ;
// Assume 'employeeSchema' and 'davidRecord' are defined

// The table is currently configured to use Parquet by default
// (e.g., table property 'write.format.default' = 'parquet')

FileWriterFactory<Record> writerFactory = GenericFileWriterFactory.builderFor(employeesTable)
    .build();

// ... (code to create OutputFile) ...
// OutputFile outputFile = ... ;
// org.apache.iceberg.encryption.EncryptedOutputFile encryptedOutputFile = ...;
// PartitionSpec spec = employeesTable.spec();
// StructLike partitionData = null;

DataWriter<Record> dataWriter = writerFactory.newDataWriter(encryptedOutputFile, spec, partitionData);

// Your code just uses the generic DataWriter:
try (DataWriter<Record> writer = dataWriter) {
    writer.add(davidRecord); // You add your generic Record
    // ... writer.result() ...
} catch (java.io.IOException e) { e.printStackTrace(); }
```
When `writerFactory.newDataWriter(...)` is called:
*   The `GenericFileWriterFactory` (actually, its parent `BaseFileWriterFactory`) checks the `employeesTable` properties. It finds that the default data file format is Parquet.
*   It then internally selects and configures a **Parquet-specific writer** that knows how to take your generic `Record` objects and write them into a Parquet file.
*   This Parquet-specific writer is wrapped in the standard `DataWriter<Record>` interface that's returned to you.

**Now, the magic:**
Imagine an administrator changes the table's default format:
```
-- SQL to change table property
ALTER TABLE employeesTable SET TBLPROPERTIES ('write.format.default'='orc');
```
If you run your *exact same Java code again*, the `GenericFileWriterFactory` will now see that the default is ORC. It will automatically pick and configure an **ORC-specific writer**. Your code (`writer.add(davidRecord)`) remains unchanged but now writes an ORC file!

This is the power of the abstraction layer. Your application logic is decoupled from the specific file format details.

**2. Reading Data (Example Revisited from Chapter 2)**

Similarly, when you read data using `IcebergGenerics.read(table).build()` as shown in [Chapter 2: Generic Data Reading](02_generic_data_reading_.md), the `GenericReader` handles this.

```java
// Conceptual part of GenericReader.openFile (or openPhysicalFile from Chapter 2)
// FileScanTask task; // This task tells us the file's path and its format (e.g., AVRO)
// InputFile input = io.newInputFile(task.file().path().toString());
// Schema fileProjection = ...;

CloseableIterable<Record> records;
switch (task.file().format()) {
    case AVRO:
        records = Avro.read(input).project(fileProjection) /* ... more config ... */ .build();
        break;
    case PARQUET:
        records = Parquet.read(input).project(fileProjection) /* ... more config ... */ .build();
        break;
    case ORC:
        records = ORC.read(input).project(fileProjection) /* ... more config ... */ .build();
        break;
    default:
        throw new UnsupportedOperationException("Cannot read " + task.file().format());
}
// These 'records' are then processed further (e.g., by DeleteFilter)
```
The `GenericReader` looks at each `FileScanTask`. The task contains information about the data file, including its `FileFormat` (e.g., `PARQUET`, `AVRO`, `ORC`).
*   Based on this format, it uses the correct "specialized tool" from Iceberg's arsenal:
    *   `org.apache.iceberg.avro.Avro.read(...)` for Avro files.
    *   `org.apache.iceberg.parquet.Parquet.read(...)` for Parquet files.
    *   `org.apache.iceberg.orc.ORC.read(...)` for ORC files.
*   Each of these tools knows how to read its specific format and produce generic `Record` objects.

You, the user, just iterate over the `CloseableIterable<Record>` and don't need to care if one file was Parquet and another was Avro.

## Under the Hood: The "Choosing the Tool" Mechanism

Let's look at how Iceberg "chooses the right tool" when you ask to write a new data file.

**Simplified Flow for Writing:**

```mermaid
sequenceDiagram
    participant You as Your Application
    participant GFWF as GenericFileWriterFactory
    participant BaseFWF as BaseFileWriterFactory
    participant IcebergParquet as org.apache.iceberg.parquet.Parquet
    participant ParquetAppender as Parquet FileAppender<Record>
    participant OutputFile

    You->>GFWF: builderFor(table).build()
    Note over GFWF: Reads table properties (e.g., default file format is PARQUET)
    You->>GFWF: newDataWriter(encryptedOutFile, spec, partition)
    GFWF->>BaseFWF: newDataWriter(encryptedOutFile, spec, partition)
    BaseFWF->>BaseFWF: Checks dataFileFormat (it's PARQUET)
    BaseFWF->>IcebergParquet: Parquet.writeData(encryptedOutFile)
    IcebergParquet-->>BaseFWF: Returns Parquet.DataWriteBuilder
    Note over BaseFWF: Calls GFWF to configure the builder
    GFWF->>Parquet.DataWriteBuilder: builder.createWriterFunc(GenericParquetWriter::create)
    BaseFWF->>Parquet.DataWriteBuilder: builder.build()
    Parquet.DataWriteBuilder-->>ParquetAppender: Creates Parquet FileAppender
    BaseFWF-->>GFWF: Returns DataWriter (wrapping ParquetAppender)
    GFWF-->>You: Returns DataWriter<Record>
    You->>DataWriter: add(myRecord)
    DataWriter->>ParquetAppender: add(myRecord)
    ParquetAppender->>OutputFile: Writes record in Parquet format
end
```

1.  **You request a `DataWriter`:** Your code calls `newDataWriter(...)` on the `GenericFileWriterFactory`.
2.  **Delegation:** `GenericFileWriterFactory` inherits from `BaseFileWriterFactory`. The call is often handled by `BaseFileWriterFactory.newDataWriter(...)`.
3.  **Check Format:** `BaseFileWriterFactory` knows the `dataFileFormat` for this operation (e.g., `PARQUET`, derived from table properties).
4.  **The Switch!** Inside `BaseFileWriterFactory.java` (which we saw a snippet of in [Chapter 4: Delete Filter & Loader](04_delete_filter___loader_.md)'s context), there's a switch statement:

    ```java
    // Simplified from BaseFileWriterFactory.java
    // ...
    // switch (dataFileFormat) {
    //     case AVRO:
    //         Avro.DataWriteBuilder avroBuilder = Avro.writeData(file) /* ... config ... */;
    //         configureDataWrite(avroBuilder); // Allows GenericFileWriterFactory to set Generic Avro writer
    //         return avroBuilder.build();
    //
    //     case PARQUET:
    //         Parquet.DataWriteBuilder parquetBuilder = Parquet.writeData(file) /* ... config ... */;
    //         configureDataWrite(parquetBuilder); // Sets GenericParquetWriter::create
    //         return parquetBuilder.build();
    //
    //     case ORC:
    //         ORC.DataWriteBuilder orcBuilder = ORC.writeData(file) /* ... config ... */;
    //         configureDataWrite(orcBuilder); // Sets GenericOrcWriter::buildWriter
    //         return orcBuilder.build();
    //
    //     default: // ...
    // }
    // ...
    ```
    This `switch` is the heart of the abstraction layer for writing. It directs the request to the correct static "entry point" for the chosen file format:
    *   `org.apache.iceberg.avro.Avro.writeData(...)`
    *   `org.apache.iceberg.parquet.Parquet.writeData(...)`
    *   `org.apache.iceberg.orc.ORC.writeData(...)`

5.  **Configure for Generic `Record`s:** Each of these (e.g., `Parquet.writeData(...)`) returns a format-specific builder (like `Parquet.DataWriteBuilder`). The `BaseFileWriterFactory` then calls an abstract method (like `configureDataWrite(parquetBuilder)`), which is implemented by `GenericFileWriterFactory`. This implementation tells the format-specific builder *how* to write generic `Record` objects. For example, for Parquet:

    ```java
    // In GenericFileWriterFactory.java
    // @Override
    // protected void configureDataWrite(Parquet.DataWriteBuilder builder) {
    //     builder.createWriterFunc(GenericParquetWriter::create);
    // }
    ```
    This line tells the Parquet machinery: "When you need to actually create the low-level writer, use the `GenericParquetWriter.create` method." This method knows how to handle `Record` objects.

6.  **Build and Return:** The format-specific builder then `build()`s the actual file writer (a `FileAppender<Record>`), which is wrapped in a `DataWriter<Record>` and returned to your application.

A similar `switch` statement exists in `GenericReader` (as shown before) for selecting the correct reader based on the `FileFormat` of the file being read. The classes `Avro`, `Parquet`, and `ORC` (from `org.apache.iceberg.avro`, `org.apache.iceberg.parquet`, and `org.apache.iceberg.orc` packages respectively) are the main entry points for Iceberg's interactions with these file formats.

**The "Specialized Tools" - `iceberg-avro`, `iceberg-parquet`, `iceberg-orc`**

Iceberg is modular. The core Iceberg library (`iceberg-api` and `iceberg-core`) defines the interfaces and the generic logic. The specific implementations for handling Parquet, Avro, and ORC files reside in separate modules:
*   `iceberg-parquet`: Contains all the code for reading and writing Parquet files, including classes like `org.apache.iceberg.parquet.Parquet` and `GenericParquetWriter`.
*   `iceberg-avro`: For Avro files, with classes like `org.apache.iceberg.avro.Avro` and `org.apache.iceberg.data.avro.DataWriter` (an Avro-specific writer for generic `Record`s).
*   `iceberg-orc`: For ORC files, with classes like `org.apache.iceberg.orc.ORC` and `GenericOrcWriter`.

When you use Iceberg, you include these modules as dependencies if you need support for those file formats. The abstraction layer ensures that the core logic can use these specialized tools without being tightly coupled to them.

## Conclusion

The File Format Abstraction Layer is a cornerstone of Iceberg's flexibility. It acts like a universal adapter, allowing your application to interact with data using generic interfaces while Iceberg handles the specifics of different file formats (Avro, Parquet, ORC) behind the scenes.

You've learned that:
*   Iceberg can read and write data in multiple file formats.
*   Your application code can remain largely format-agnostic, thanks to this abstraction.
*   Components like `GenericFileWriterFactory` (via `BaseFileWriterFactory`) and `GenericReader` use `switch` statements or similar logic to select the correct format-specific "tool" (e.g., `Parquet.writeData()`, `Avro.read()`).
*   These specialized tools are provided by modules like `iceberg-parquet`, `iceberg-avro`, and `iceberg-orc`.

This design makes your Iceberg applications more robust to changes in underlying storage formats and simpler to develop.

So far, we've focused on individual rows and files. But Iceberg tables can also be partitioned. How does Iceberg manage statistics related to these partitions, which is crucial for query planning? We'll explore this in the next chapter: [Chapter 6: Partition Statistics Handler](06_partition_statistics_handler_.md).

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)