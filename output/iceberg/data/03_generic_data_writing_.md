# Chapter 3: Generic Data Writing

In [Chapter 2: Generic Data Reading](02_generic_data_reading_.md), we learned how Iceberg acts like a smart librarian, helping us read data from tables as generic `Record` objects without worrying about the underlying file formats or complexities. Now, what if we want to do the opposite? How do we add new information—new rows of data—into our Iceberg table? That's where "Generic Data Writing" comes in!

## The Publishing House: Adding New Information

Imagine you've written a new chapter for a book (your new data) and you want to add it to the main manuscript (your Iceberg table). You wouldn't just staple your pages in; you'd go to a publishing house.

This "publishing house" is what Generic Data Writing in Iceberg represents.
*   It takes your manuscript (your data in the form of generic `Record` objects, which we learned about in [Chapter 1: Generic Row Representation (`Record` & `InternalRecordWrapper`)](01_generic_row_representation___record_____internalrecordwrapper___.md)).
*   It chooses the right printing press (the file format, like Parquet, Avro, or ORC, based on your table's settings).
*   It organizes the pages correctly (ensuring the data matches the table's schema).
*   It produces a new, properly formatted section of the book (a data file).
*   Finally, it tells the main library catalog that a new section is available.

**Our Use Case:** Let's say we have our `employeesTable` and we've just hired a new employee:
*   ID: `103`
*   Name: `"David"`
*   Hire Date: `March 10, 2024`
*   Department: `"Data Science"`

We want to add David's information as a new row into our `employeesTable`. We have this data as a `Record` object, and we want Iceberg to handle the rest.

## Meet the Key Players: `FileWriterFactory` and `DataWriter`

Two main components help us in this "publishing" process:

1.  **`FileWriterFactory<T>` (The Publishing House Manager):**
    *   This is like the manager of the publishing house. You tell it what kind of "book" (data file) you want to create (for which table, and potentially what format).
    *   It knows how to set up the right "printing press" (`DataWriter`) for your needs.
    *   Specifically, we'll often use `GenericFileWriterFactory`, which is designed to work with our generic `Record` objects.

2.  **`DataWriter<T>` (The Printing Press):**
    *   Once the factory gives you a `DataWriter`, this is your actual "printing press."
    *   You feed your `Record` objects (your manuscript pages) to the `DataWriter` one by one.
    *   It takes care of writing these records into a data file in the correct format (e.g., Parquet) and according to the table's schema.
    *   When you're done writing, you "close" the writer, and it finalizes the data file, giving you back a `DataFile` object. This `DataFile` object is like a receipt, containing information about the file you just wrote (path, size, metrics, etc.).

## Adding Data: Step-by-Step

Let's add David's record to our hypothetical `employeesTable`.

**Prerequisites:**
*   An existing Iceberg `Table` object (e.g., `employeesTable`).
*   The schema of the table.
*   An `OutputFile` which tells Iceberg *where* to write the new data file. Iceberg's `OutputFileFactory` can help create these, often placing them in unique, temporary locations within your table's data directory.

**Step 1: Prepare Your Data as `Record`s**

First, let's create a `GenericRecord` for David. Assume our `employeesTable` has the schema: `id (int), name (string), hire_date (date), department (string)`.

```java
import org.apache.iceberg.Schema;
import org.apache.iceberg.types.Types;
import org.apache.iceberg.data.GenericRecord;
import java.time.LocalDate;

// Define the schema (same as in Chapter 1, with an added department)
Schema employeeSchema = new Schema(
    Types.NestedField.required(1, "id", Types.IntegerType.get()),
    Types.NestedField.optional(2, "name", Types.StringType.get()),
    Types.NestedField.required(3, "hire_date", Types.DateType.get()),
    Types.NestedField.optional(4, "department", Types.StringType.get())
);

// Create a GenericRecord for David
GenericRecord davidRecord = GenericRecord.create(employeeSchema);
davidRecord.setField("id", 103);
davidRecord.setField("name", "David");
davidRecord.setField("hire_date", LocalDate.of(2024, 3, 10));
davidRecord.setField("department", "Data Science");

// You could have a list of many such records to write
// List<Record> newEmployees = Arrays.asList(davidRecord, anotherRecord, ...);
```
Here, we're using the `GenericRecord` we learned about in [Chapter 1: Generic Row Representation (`Record` & `InternalRecordWrapper`)](01_generic_row_representation___record_____internalrecordwrapper___.md). The `hire_date` is a `LocalDate`, and Iceberg's writers (with the help of mechanisms like `InternalRecordWrapper` internally) will convert it to the appropriate format for storage.

**Step 2: Get a `FileWriterFactory`**

Iceberg provides `GenericFileWriterFactory` for writing `Record` objects. We can build one for our table.

```java
import org.apache.iceberg.Table;
import org.apache.iceberg.data.GenericFileWriterFactory;
import org.apache.iceberg.io.FileWriterFactory;
import org.apache.iceberg.FileFormat; // Assuming you might want to specify

// Assume 'employeesTable' is an existing Iceberg Table object
// Table employeesTable = ... ;

// Create a FileWriterFactory.
// It will infer schema, sort order, and default file format from the table.
FileWriterFactory<Record> writerFactory = GenericFileWriterFactory.builderFor(employeesTable)
    // Optionally, you can override the default file format for data files
    // .dataFileFormat(FileFormat.PARQUET)
    .build();
```
This factory is now configured based on `employeesTable`'s properties, like its schema and default file format (e.g., Parquet).

**Step 3: Create an `OutputFile` and a `DataWriter`**

We need to tell Iceberg where to physically write the new data file. An `OutputFileFactory` associated with the table can generate a unique path for us. Then, we use our `writerFactory` to get a `DataWriter`.

```java
import org.apache.iceberg.io.OutputFile;
import org.apache.iceberg.io.DataWriter;
import org.apache.iceberg.DataFile;
import org.apache.iceberg.WriteResult;
import org.apache.iceberg.PartitionSpec; // For partition information
import org.apache.iceberg.StructLike;   // For partition data

// For unpartitioned tables, spec is PartitionSpec.unpartitioned()
// and partitionData can be null or an empty StructLike.
PartitionSpec spec = employeesTable.spec();
StructLike partitionData = null; // Assuming unpartitioned for simplicity
                                 // For a partitioned table, this would represent the partition value.

// Create an OutputFile for the new data.
// The OutputFileFactory helps create unique file names in the table's data location.
OutputFile outputFile = employeesTable.io().newOutputFile(
    spec.specId() + "-0-" + java.util.UUID.randomUUID().toString() + "-0.parquet" // Example unique name
);

// Get a DataWriter from the factory for this output file
// We also need to provide an EncryptedOutputFile, PartitionSpec, and partition data
// For simplicity, we assume plain (non-encrypted) files
org.apache.iceberg.encryption.EncryptedOutputFile encryptedOutputFile =
    org.apache.iceberg.encryption.EncryptionUtil.plainAsEncryptedOutput(outputFile);

DataWriter<Record> dataWriter = writerFactory.newDataWriter(encryptedOutputFile, spec, partitionData);
```
The `newDataWriter` method takes the `EncryptedOutputFile` (even if not actually encrypted), the table's `PartitionSpec`, and the specific `partitionData` this file belongs to (if the table is partitioned).

**Step 4: Write Your Records**

Now, feed your `Record` objects to the `dataWriter`.

```java
try (DataWriter<Record> writer = dataWriter) { // Use try-with-resources for auto-closing
    writer.add(davidRecord);
    // If you had a list of records:
    // for (Record record : newEmployees) {
    //     writer.add(record);
    // }

    // When done, the writer.close() in try-with-resources will be called.
    // This finalizes the file and returns a WriteResult.
    WriteResult result = writer.result();
    DataFile dataFile = result.dataFiles()[0]; // Get the DataFile object

    System.out.println("Successfully wrote data file: " + dataFile.path());
    System.out.println("Record count: " + dataFile.recordCount());

    // NEXT STEP: Commit this dataFile to the table
    // employeesTable.newAppend().appendFile(dataFile).commit();

} catch (java.io.IOException e) {
    e.printStackTrace();
}
```
**Output (Example):**
```
Successfully wrote data file: s3://my-bucket/my_table/data/0-0-some-uuid-0.parquet
Record count: 1
```
When you call `add()`, the `dataWriter` writes the `Record` to the `outputFile` in the chosen format. The `try-with-resources` ensures `writer.close()` is called. Closing the writer finalizes the file on disk and returns a `WriteResult`, which contains one or more `DataFile` objects. Each `DataFile` holds metadata about the file just written (path, format, record count, metrics, etc.).

**Important Next Step: Committing the DataFile**
Just writing a data file isn't enough for it to be part of your Iceberg table. You need to tell Iceberg about this new file by committing it to the table's metadata. This is typically done using:
```java
// After getting the 'dataFile' from the writer.result():
// employeesTable.newAppend()
//     .appendFile(dataFile)
//     .commit();
```
This "commit" step updates the table's manifest list and snapshot, making the new data visible to queries. Our current chapter focuses only on the *writing of the file itself*.

## What Happens Under the Hood?

When you use `GenericFileWriterFactory` and `DataWriter`, Iceberg handles several things for you:

```mermaid
sequenceDiagram
    participant User
    participant GFWF as GenericFileWriterFactory
    participant ParquetDataWriterBuilder as Parquet.DataWriteBuilder
    participant IcebergDataWriter as org.apache.iceberg.io.DataWriter
    participant OutputFile

    User->>GFWF: builderFor(employeesTable).build()
    GFWF-->>User: GenericFileWriterFactory instance
    User->>GFWF: newDataWriter(encryptedOutFile, spec, partition)
    Note over GFWF: Determines format (e.g., Parquet from table props)
    Note over GFWF: Gets dataSchema from table
    GFWF->>ParquetDataWriterBuilder: Parquet.writeData(encryptedOutFile).schema(dataSchema)...
    Note over ParquetDataWriterBuilder: Sets up Parquet specific writer using GenericParquetWriter.create
    ParquetDataWriterBuilder-->>GFWF: Configured Parquet Writer (FileAppender<Record>)
    GFWF-->>IcebergDataWriter: Creates DataWriter (wrapping the Parquet FileAppender)
    IcebergDataWriter-->>User: DataWriter instance
    User->>IcebergDataWriter: add(davidRecord)
    IcebergDataWriter->>OutputFile: Writes David's record (Parquet format)
    User->>IcebergDataWriter: close()
    IcebergDataWriter->>OutputFile: Finalizes Parquet file
    IcebergDataWriter-->>User: WriteResult (containing DataFile for davidRecord's file)
end
```

1.  **Factory Configuration (`GenericFileWriterFactory.builderFor(table)`):**
    *   The builder inspects the `employeesTable` to find its schema (`dataSchema`), default data file format (e.g., `PARQUET` from `table.properties().get(DEFAULT_FILE_FORMAT)`), sort order, etc.

2.  **Requesting a Writer (`factory.newDataWriter(...)`):**
    *   The `GenericFileWriterFactory` (which extends `BaseFileWriterFactory`) receives this request.
    *   The `BaseFileWriterFactory.newDataWriter()` method (see `BaseFileWriterFactory.java`) has a `switch` statement based on the `dataFileFormat` (e.g., Parquet, Avro, ORC).
    *   It calls the appropriate static method from Iceberg's file format libraries, for example, `Parquet.writeData(encryptedOutputFile)`.
    *   This returns a format-specific builder (e.g., `Parquet.DataWriteBuilder`).

3.  **Configuring the Format-Specific Writer:**
    *   The `BaseFileWriterFactory` then calls an abstract method like `configureDataWrite(parquetBuilder)`.
    *   `GenericFileWriterFactory.java` implements this method. For Parquet, it does:
        ```java
        // In GenericFileWriterFactory.java
        @Override
        protected void configureDataWrite(Parquet.DataWriteBuilder builder) {
            builder.createWriterFunc(GenericParquetWriter::create);
        }
        ```
        This tells the Parquet builder to use `GenericParquetWriter.create` as the function to actually create the writer instance that can handle generic `Record` objects. Similar configurations exist for Avro (`DataWriter::create`) and ORC (`GenericOrcWriter::buildWriter`).
    *   The builder is then configured with the schema, table properties, sort order, etc.

4.  **Building the Writer:**
    *   The format-specific builder's `build()` method is called. This returns a `FileAppender<Record>` (for Parquet, Avro, or ORC) that knows how to take `Record` objects and write them in its specific format.
    *   This `FileAppender<Record>` is then wrapped in an `org.apache.iceberg.io.DataWriter<Record>` object, which is what's returned to you.

5.  **Writing Records (`dataWriter.add(record)`):**
    *   When you call `add(record)` on the `DataWriter`, it passes the `Record` to the underlying format-specific `FileAppender`.
    *   The `FileAppender` handles the details of serializing the `Record` fields according to the schema and writing them to the output file. This is where type conversions (like `LocalDate` to an integer representation for Parquet dates) happen, often leveraging internal mechanisms similar to the `InternalRecordWrapper` we saw in Chapter 1.

6.  **Closing the Writer (`dataWriter.close()`):**
    *   This flushes any buffered data to the file, writes file footers (e.g., Parquet metadata), closes the file stream, and collects statistics about the written file (record count, column metrics, etc.).
    *   It then packages this information into a `DataFile` object, which is part of the `WriteResult`.

This entire process abstracts away the complexities of specific file formats. You, as the user, just work with generic `Record` objects and a standard `DataWriter` interface, and Iceberg's factories ensure the correct "printing press" is used. The choice of file format and other write properties are typically controlled by the table's settings, making the write path consistent.

## Conclusion

Generic Data Writing in Iceberg provides a powerful yet simple way to add new information to your tables. By using `GenericFileWriterFactory` and `DataWriter`, you can feed your generic `Record` objects to Iceberg, and it takes care of:
*   Choosing the correct file format (Parquet, Avro, ORC).
*   Writing the data according to the table's schema.
*   Producing a `DataFile` object that describes the newly written file.

Remember, writing the data file is the first major step. The second is committing this `DataFile` to the Iceberg table's metadata so it becomes visible in queries.

Now that we know how to read and write data files, what about modifying existing data? Iceberg handles deletes in a special way. We'll explore this in the next chapter on [Chapter 4: Delete Filter & Loader](04_delete_filter___loader_.md).

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)