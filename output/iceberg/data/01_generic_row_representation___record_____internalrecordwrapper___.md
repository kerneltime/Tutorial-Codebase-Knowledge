# Chapter 1: Generic Row Representation (`Record` & `InternalRecordWrapper`)

Welcome to your first step into the world of Apache Iceberg's data handling! In this chapter, we'll explore how Iceberg represents a single row of data in a flexible way and how it cleverly handles different data types.

## What's the Big Deal with Data Rows?

Imagine you have a spreadsheet full of information, say, about your favorite books. Each row in that spreadsheet represents one book and has columns like "Title," "Author," and "Publication Date."

In the world of big data and Iceberg, we deal with similar "rows" of data, but on a much larger scale. We need a standard, easy-to-use way to work with these rows, no matter where the data comes from or where it's going.

**Our Use Case:** Let's say we have information about an employee: their ID, name, and hiring date.
*   ID: `101`
*   Name: `"Alice"`
*   Hire Date: `January 15, 2023` (represented in Java as a `LocalDate` object)

When we want to save this information into an Iceberg table, Iceberg might need to store that `Hire Date` in a special internal format – for example, as the number of days since a standard reference point (like January 1, 1970). How do we make this conversion smooth and invisible?

This is where `Record` and `InternalRecordWrapper` come into play!

## Meet the Key Players: `StructLike`, `Record`, and `InternalRecordWrapper`

Let's break down these important concepts.

### 1. `StructLike`: The Blueprint for a Row

*   **What it is:** Think of `StructLike` as Iceberg's general "idea" or "contract" for what any row of data should look like. It's an interface, meaning it defines *what* a row can do (like tell you how many fields it has, or give you the value of a specific field) but not *how* it stores that data.
*   **Analogy:** If you're building with LEGOs, `StructLike` is like the rule that says "anything you build must be able to connect to other LEGO pieces." It doesn't tell you whether to build a car or a spaceship, just that it needs to follow the basic LEGO connection rules.

### 2. `Record`: Your Handy Data Box

*   **What it is:** `Record` is Iceberg's standard, concrete way to represent a row of data. It's a class that actually implements the `StructLike` "contract." You can create a `Record` object and fill it with your data.
*   **Analogy:** If `StructLike` is the general idea of a "lunch container," then `Record` is like an actual bento box. It has specific compartments where you can put your rice, vegetables, and protein. You can easily create a `Record` and put your data (like Alice's ID, name, and hire date) into its "compartments."

Iceberg provides `GenericRecord` as a common implementation of `Record`. You'd typically use `GenericRecord.create(schema)` to make one, where `schema` describes the structure of your data (like column names and types).

### 3. `InternalRecordWrapper`: The Smart Type Translator

*   **What it is:** The `InternalRecordWrapper` is a very clever helper. It takes a `Record` (or any `StructLike` object) and helps convert specific data types into the internal formats Iceberg expects, especially when writing data to files. It can also do the reverse when reading data.
*   **Analogy:** Imagine you're traveling to a country that uses a different currency. You have your home currency (like our `LocalDate` for the hire date). The `InternalRecordWrapper` is like a currency exchange booth at the airport. You give it your `LocalDate`, and it gives you back the equivalent value in Iceberg's "local currency" (e.g., the number of days since epoch). It ensures your data is in the right format for Iceberg.

It's like a universal translator for individual data fields within a row, ensuring type compatibility!

## Putting Them to Work: Our Employee Example

Let's see how we'd use these tools for our employee, Alice.

First, we need to tell Iceberg what our data looks like. This is called a **schema**.

```java
import org.apache.iceberg.Schema;
import org.apache.iceberg.types.Types;
import org.apache.iceberg.data.GenericRecord;
import org.apache.iceberg.data.InternalRecordWrapper;
import java.time.LocalDate;

// 1. Define the schema for our employee data
Schema employeeSchema = new Schema(
    Types.NestedField.required(1, "id", Types.IntegerType.get()),
    Types.NestedField.optional(2, "name", Types.StringType.get()),
    Types.NestedField.required(3, "hire_date", Types.DateType.get())
);
```
This schema tells Iceberg we have three fields: an integer `id`, a string `name`, and a `date` called `hire_date`.

Next, let's create a `Record` to hold Alice's data:

```java
// 2. Create a GenericRecord for Alice
GenericRecord aliceRecord = GenericRecord.create(employeeSchema);
aliceRecord.setField("id", 101);
aliceRecord.setField("name", "Alice");
aliceRecord.setField("hire_date", LocalDate.of(2023, 1, 15)); // Original Java LocalDate

// At this point, aliceRecord.getField("hire_date") would give you back the LocalDate object.
// System.out.println("Original hire_date: " + aliceRecord.getField("hire_date"));
// Output: Original hire_date: 2023-01-15
```
Here, we create a `GenericRecord` based on our `employeeSchema` and fill it with Alice's details. The `hire_date` is stored as a standard Java `LocalDate`.

Now, let's use the `InternalRecordWrapper` to see how it prepares the `hire_date` for Iceberg's internal storage:

```java
// 3. Create an InternalRecordWrapper for our schema
InternalRecordWrapper wrapper = new InternalRecordWrapper(employeeSchema.asStruct());

// 4. Wrap Alice's record
wrapper.wrap(aliceRecord);

// 5. Access the hire_date through the wrapper
// Iceberg's DateType is stored as days since epoch (1970-01-01)
// LocalDate.of(2023, 1, 15) is 19371 days since 1970-01-01.
Integer internalDate = wrapper.get(2, Integer.class); // Field at index 2 is 'hire_date'

// System.out.println("Internal representation of hire_date: " + internalDate);
// Output: Internal representation of hire_date: 19371
```
When we access `hire_date` (which is the field at position 2) through the `wrapper`, we don't get the `LocalDate` object back. Instead, we get an `Integer`: `19371`. This is the number of days between January 1, 1970, and January 15, 2023. The `InternalRecordWrapper` automatically performed this conversion because our schema defined `hire_date` as an Iceberg `DateType`.

This is super helpful because when Iceberg writes this data to a file (like Parquet or ORC), it needs the date in this "days since epoch" integer format. The wrapper handles this transparently!

## Under the Hood: How Does `InternalRecordWrapper` Work?

You might be wondering how this "magic" happens. Let's peek inside.

**The Setup Phase (When you create an `InternalRecordWrapper`):**

1.  When you create an `InternalRecordWrapper` and give it a schema (like our `employeeSchema`), it examines each field in that schema.
2.  For field types that need special handling (like `Date`, `Time`, `Timestamp`), it prepares a little "converter function." For example, for a `DateType`, it knows it needs a function that can turn a `LocalDate` into an integer (days since epoch).
3.  For fields that don't need conversion (like our `id` (Integer) or `name` (String) in this example), it doesn't set up a special converter.

**The "Get" Phase (When you call `wrapper.get(...)`):**

Here's a simplified step-by-step of what happens when you ask the wrapper for data:

```mermaid
sequenceDiagram
    participant You
    participant Wrapper as InternalRecordWrapper
    participant AliceRecord as GenericRecord
    participant DateConverter as "Converter Function (for Date)"
    participant DateTimeUtil as "Iceberg's DateTime Helper"

    You->>Wrapper: get(2, Integer.class) (for hire_date)
    Wrapper->>Wrapper: Has a converter for field at pos 2? (Yes, for Date)
    Wrapper->>AliceRecord: get(2, Object.class) (get original value)
    AliceRecord-->>Wrapper: Returns LocalDate.of(2023, 1, 15)
    Wrapper->>DateConverter: apply(LocalDate.of(2023, 1, 15))
    DateConverter->>DateTimeUtil: daysFromDate(LocalDate.of(2023, 1, 15))
    DateTimeUtil-->>DateConverter: Returns 19371
    DateConverter-->>Wrapper: Returns 19371
    Wrapper-->>You: Returns 19371 (as Integer)
end
```

1.  You call `wrapper.get(fieldPosition, expectedJavaClass)`.
2.  The `InternalRecordWrapper` checks if it has a pre-configured converter function for the field at `fieldPosition`.
3.  **If a converter exists:**
    a.  It first retrieves the original value from the wrapped `Record` (e.g., the `LocalDate` object).
    b.  It then passes this original value to the specific converter function.
    c.  The converter function does its job (e.g., `DateTimeUtil.daysFromDate(localDate)` turns the `LocalDate` into an integer).
    d.  The wrapper returns this converted value.
4.  **If no converter exists** (e.g., for an Integer or String field that doesn't need special internal conversion):
    a.  It simply retrieves the value directly from the wrapped `Record` and returns it as is.

### A Glimpse into the Code (`InternalRecordWrapper.java`)

Let's look at some simplified parts of the `InternalRecordWrapper.java` code to understand this better.

**1. The Constructor: Preparing the Converters**

When an `InternalRecordWrapper` is created, its constructor loops through the fields of the provided schema and figures out if a converter is needed for each one.

```java
// Simplified from InternalRecordWrapper.java
public InternalRecordWrapper(Types.StructType struct) {
    // 'transforms' is an array to hold our converter functions
    this.transforms = new Function[struct.fields().size()]; // Function for each field

    for (int i = 0; i < struct.fields().size(); i++) {
        Types.NestedField field = struct.fields().get(i);
        // For each field, call 'converter' to get the right conversion function
        this.transforms[i] = converter(field.type());
    }
}
```
This code creates an array called `transforms`. Each element in this array can hold a function. It then calls a helper method `converter()` for each field type in your schema to get the appropriate conversion function (if any) and stores it in the `transforms` array.

**2. The `converter()` Method: Deciding How to Convert**

This static method is like a decision-maker. Based on the field type, it returns the correct conversion logic.

```java
// Simplified from InternalRecordWrapper.java
private static Function<Object, Object> converter(Type type) {
    switch (type.typeId()) {
        case DATE:
            // If the type is DATE, return a function that converts
            // a LocalDate object to an int (days since epoch).
            return date -> DateTimeUtil.daysFromDate((LocalDate) date);
        case TIME:
            // If the type is TIME, return a function that converts
            // a LocalTime object to a long (microseconds since midnight).
            return time -> DateTimeUtil.microsFromTime((LocalTime) time);
        // ... other cases for TIMESTAMP, FIXED, STRUCT, etc.
        default:
            // If no special conversion is needed, return null.
            return null;
    }
}
```
You can see that for `DATE`, it uses `DateTimeUtil.daysFromDate()`. For `TIME`, it uses `DateTimeUtil.microsFromTime()`. If a type doesn't need conversion, it returns `null`, meaning "no special action needed."

**3. The `get()` Method: Applying the Conversion**

This is where the actual conversion happens when you request data.

```java
// Simplified from InternalRecordWrapper.java
@Override
public <T> T get(int pos, Class<T> javaClass) {
    // Is there a converter function stored for this field position ('pos')?
    if (transforms[pos] != null) {
        Object originalValue = wrapped.get(pos, Object.class); // Get original from wrapped record
        if (originalValue == null) {
            return null; // Handle nulls directly
        }
        // Yes, there's a converter! Apply it.
        Object convertedValue = transforms[pos].apply(originalValue);
        return javaClass.cast(convertedValue); // Return the converted value
    }

    // No converter for this field, just get the value directly.
    return wrapped.get(pos, javaClass);
}
```
When you call `get(pos, ...)`, it checks `transforms[pos]`. If it's not `null` (meaning there's a converter), it gets the original value from the `wrapped` record, applies the `transforms[pos]` function to it, and returns the result. Otherwise, it just fetches the value directly from the `wrapped` record.

The `InternalRecordWrapper` also has a handy `wrap(StructLike record)` method that lets you reuse the same wrapper instance with different `Record` objects of the same schema, and a `copyFor(StructLike record)` to create a new wrapper instance configured the same way but for a new record.

One important note: the `set(int pos, T value)` method in `InternalRecordWrapper` will throw an `UnsupportedOperationException`. This means you can't use the wrapper to change data in the underlying record; it's primarily for reading and transforming data on the fly.

## Conclusion

Phew! We've covered a lot. Let's recap:

*   Iceberg uses `StructLike` as a general interface for row-like data.
*   `Record` (often `GenericRecord`) is Iceberg's standard, easy-to-use container for holding a row of data. You can think of it as a generic holder for your data fields.
*   `InternalRecordWrapper` is a powerful utility that "wraps" around a `Record` (or any `StructLike`). Its main job is to transparently convert data types (like dates, times, timestamps) between their common Java representations and the specific internal formats Iceberg uses for storage. This is crucial for writing data correctly into Iceberg files.

Understanding `Record` and `InternalRecordWrapper` is fundamental because they ensure that data, regardless of its original Java type, is correctly interpreted and formatted according to the Iceberg table's schema, especially when preparing data to be written.

Now that we understand how Iceberg can represent a single row of data and handle its types, let's see how we can read many such rows from data files in the next chapter on [Generic Data Reading](02_generic_data_reading_.md).

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)