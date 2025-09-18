# Chapter 5: Iterators

In the [previous chapter](04_versioning_and_point_in_time_views__superversion__.md), we learned how `db` uses `SuperVersion` to provide a "clear photograph" or a stable, point-in-time snapshot of the database. This is essential for getting consistent reads while writes and background maintenance are happening.

Now, let's talk about how we actually *read* data. We know how to get a single value for a single key using `Get()`. But what if we want to answer questions like:
*   "Find all users whose name starts with 'J'."
*   "Show me the 10 most recent log entries from this morning."
*   "How many items do we have in the 'products' category?"

These queries require scanning over a *range* of keys, not just fetching one. For this, `db` provides a powerful tool: the **Iterator**.

### The Challenge: A Unified View of a Messy Room

Imagine you're looking for all your blue socks. Some are in your active laundry basket (`MemTable`), some are in a clean pile on your bed waiting to be folded (immutable `MemTable`s), and the rest are neatly put away in different drawers (SST files on disk). To find all your blue socks, you'd have to look in all these places.

This is exactly the problem an iterator solves. Your data isn't in one single place. It's spread across:
1.  The active **`MemTable`** (newest data, in memory).
2.  One or more **immutable `MemTable`s** (recently written data, in memory).
3.  Many **SST files** on disk, organized into different levels.

An iterator is like a smart assistant that knows where to look. It gives you a single, unified, sorted view of all your data, no matter where it's physically stored. It cleverly combines all these sources, hides deleted items, and ignores data that's newer than your snapshot, presenting you with a simple, clean, sequential list of key-value pairs.

### The `DBIter`: Your Smart Cursor

The main iterator implementation you use is called `DBIter`. Think of it as a **smart cursor** or a playhead on a music track. You can:
*   `Seek()` to a specific key to start.
*   Call `Next()` to move to the next key-value pair.
*   Call `Prev()` to move to the previous one.
*   Check `Valid()` to see if the cursor is pointing to a real item.
*   Read the `key()` and `value()` at the current position.

Let's see the classic way to scan all keys with a certain prefix.

```cpp
// 1. Create a new iterator. It gets a snapshot automatically.
rocksdb::Iterator* it = db->NewIterator(rocksdb::ReadOptions());

// 2. Seek to the first key that starts with "user:".
std::string start_key = "user:";
for (it->Seek(start_key); 
     it->Valid() && it->key().starts_with(start_key); 
     it->Next()) 
{
  // 3. Process the key and value.
  std::cout << it->key().ToString() << ": " << it->value().ToString() << std::endl;
}

// 4. Check for any errors during iteration.
assert(it->status().ok()); 

// 5. Clean up the iterator.
delete it;
```
This simple loop hides an incredible amount of complexity. The `DBIter` (`it`) is doing all the hard work of peeking into the `MemTable`s and all the relevant SST files for you.

### Under the Hood: The Merging Iterator

How does `DBIter` perform this magic? It doesn't look at one source at a time. That would be slow. Instead, it builds a tree of iterators, with a special **Merging Iterator** at its core.

1.  **Get a Snapshot:** When you call `NewIterator()`, `db` grabs the current [SuperVersion](04_versioning_and_point_in_time_views__superversion__.md). This locks in the exact set of `MemTable`s and SST files the iterator will look at for its entire lifetime.
2.  **Build the Iterator Tree:** `DBIter` creates a separate internal iterator for each source of data: one for the active `MemTable`, one for each immutable `MemTable`, and one for each level of the SST files.
3.  **Merge-Sort on the Fly:** It feeds all these internal iterators into a `MergingIterator`. The `MergingIterator` works like a merge-sort. To find the next overall key, it asks all its child iterators, "What's your current key?" and picks the one that is smallest.

```mermaid
graph TD
    subgraph DBIter (Your Smart Cursor)
        A[MergingIterator]
    end

    subgraph Data Sources (Defined by SuperVersion)
        B(MemTable Iterator)
        C(Immutable MemTables Iterator)
        D(Level 0 SSTs Iterator)
        E(Level 1 SSTs Iterator)
        F(...)
    end
    
    A --> B
    A --> C
    A --> D
    A --> E
    A --> F
```
When you call `Next()`, the `MergingIterator` finds the globally smallest key across all sources. But its job isn't done!

### It's More Than Just Merging

Let's say the `MergingIterator` finds that the smallest key is `"cat"` with sequence number 95. The `DBIter` then has to ask some critical questions:

*   **Is it visible?** Is the sequence number (95) less than or equal to my snapshot's sequence number? (Let's say our snapshot is 100, so yes).
*   **Is it a deletion?** What is the `ValueType`? If it's `kTypeDeletion`, then this key is dead. The `DBIter` must discard it and also remember to ignore any *older* versions of `"cat"` it might see later from other sources.
*   **Have I seen a newer version?** If the `DBIter` just showed you `"cat"` from the `MemTable` (sequence 100), and now it sees `"cat"` from an SST file (sequence 95), it knows to skip the older one.

The core of this logic lives in the `DBIter::FindNextUserEntryInternal` method in `db_iter.cc`. This is the heart of the `Next()` call.

```cpp
// Simplified from DBIter::FindNextUserEntryInternal in db_iter.cc
// This is the main loop for finding the next valid key for the user.
do {
  // 1. Get the next smallest internal key from the MergingIterator.
  ParseKey(&ikey_);

  // 2. Is this key-value version visible to our snapshot?
  if (IsVisible(ikey_.sequence, ...)) {
    
    // 3. Is this a new user key, or an older version of the one we just saw?
    if (/* this is an older version of the previous key */) {
       // Skip it, we've already seen the newest version.
       PERF_COUNTER_ADD(internal_key_skipped_count, 1);
    } else {
      // 4. This is a new user key. What type of entry is it?
      switch (ikey_.type) {
        case kTypeValue:
          // It's a live value! Save it and we're done for this Next() call.
          saved_key_.SetUserKey(ikey_.user_key, ...);
          value_ = iter_.value();
          valid_ = true;
          return true; // Exit the loop and return to the user.

        case kTypeDeletion:
          // It's a delete marker. Remember to skip all older versions of
          // this key we might see later.
          saved_key_.SetUserKey(ikey_.user_key, ...);
          skipping_saved_key = true;
          break; // Continue the loop to find the *next* user key.
      }
    }
  }
  // 5. If not visible or skipped, advance the internal MergingIterator.
  iter_.Next();
} while (iter_.Valid());
```
This loop is the engine that drives the iterator forward. It repeatedly pulls the next-smallest key from the underlying sources and runs it through this filter until it finds an entry that is valid, visible, and the newest version for that key, which it then presents to you.

The `DBIter` class itself, defined in `db_iter.h`, holds all the state needed to manage this process.

```cpp
// Simplified from db_iter.h
class DBIter final : public Iterator {
 public:
  bool Valid() const override;
  void Next() final override;
  Slice key() const override;
  Slice value() const override;
  // ... other Iterator methods ...

 private:
  // The underlying MergingIterator that combines all data sources.
  IteratorWrapper iter_;
  
  // The snapshot sequence number for this iterator's consistent view.
  SequenceNumber sequence_;
  
  // The current key and value we are presenting to the user.
  IterKey saved_key_;
  Slice value_;

  bool valid_;
  Direction direction_; // Are we going forward or backward?
  // ... and many other fields for managing state ...
};
```
This object is the complete package: it holds the state of the iteration (`saved_key_`, `valid_`), the logic to advance (`Next()`), and the connection to the underlying data sources (`iter_`) all bound to a specific point-in-time (`sequence_`).

### Conclusion

You've just learned about the primary tool for reading ranges of data from `db`.

*   An **Iterator** provides a unified, sorted view of key-value pairs.
*   It's a "smart cursor" that merges data from the **`MemTable`s** and on-disk **SST files**.
*   The main implementation, **`DBIter`**, uses a `MergingIterator` to find the next smallest key across all sources.
*   It's more than a simple merge: it respects your snapshot's **`SuperVersion`**, handles **deleted keys** (tombstones), and correctly shows only the **newest version** of any key.

We now understand how data is written to memory ([MemTable & WAL](02_in_memory_writes_and_durability__memtable___wal__.md)) and how it can be read back with iterators. But the `MemTable` can't grow forever. What happens when it gets full? It needs to be written to a file on disk.

In the next chapter, we'll explore exactly that process: flushing the `MemTable` to an SST file.

**Next**: [Chapter 6: Flushing MemTables to Disk (FlushJob)](06_flushing_memtables_to_disk__flushjob__.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)