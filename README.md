# Minimal Write-Ahead Log (WAL)

Databases rely on WALs to ensure durability and atomicity. Writing one forces to grapple with file I/O, byte serialization, append-only logs, and state reconstruction.

## Part 1: What is a WAL ?

### The durability problem: 
In-memory data structures (like hash maps or trees) are fast, but volatile. If the power cuts, data vanishes. How do we make state survive a crash without writing the entire data structure to disk on every write ? (Answer: Append-only logs)

### Log Record Anatomy:
Every entry in a WAL needs structural metadata. At minimum, a record needs:
- **LSN (Log Sequence Number):** Monotonically increasing counter for ordering
- **Operation Type:** ==INSERT==, ==DELETE==, ==UPDATE==
- **Payload:** Key and Value bytes
- **Checksum:** To detect corrupted or partially written trailing records (torn writes)

### Crash Recovery:
When a database boots up after a crash, it doesn't know what state the disk is in. It must read the WAL from byte 0 to the end, replaying every committed operation into an in-memory store to bring it back to the exact state it was in before the crash

## Part 2: Practical goals

### Milestone 1: Log serialization & disk append
- **Goal:** Define a binary or structured log format and append records to a file on disk
- **Task:**
    - Define a ==LogRecord== struct/class
    - Implement a function ==serialize(record) -> bytes== and ==deserialize(bytes) -> record==
    - Write an ==AppendOnlyLog== class that opens a file in append mode (==ab+==) and writes serialized records, ensuring you flush (==fsyn==) to disk

### Milestone 2: The write path & memtable
- **Goal:** Combine an in-memory data store with the WAL
- **Tasks:**
    - Build a simple key-value store class (==KVStore==)
    - Inside ==KVStore==, maintain an in-memory hash map or dictionary
    - Implement ==put(key, value)== and ==delete(key)==:
        1. Write the operation to the WAL first
        2. Only if the WAL write succeeds, update the in-memory hash map

### Milestone 3: Crash recovery engine
- **Goal:** Rebuild the in-memory state from disk upon startup.
- **Tasks:**
    - Write a ==recover()== method for your ==KVStore== that runs when the store initializes
    - It should open the WAL file from the beginning, read records sequentially, handle any corrupted/torn tail records gracefully, and repopulate the in-memory hash map
    - **Test it:** Write a script that inserts 10 keys, abruptly "kills" the process, instantiates a new store instance pointing to the same WAL file, and asserts that all 10 keys are present in memory.
