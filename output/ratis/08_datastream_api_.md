# Chapter 8: DataStream API

Welcome to Chapter 8! In the [previous chapter](07_snapshotting__snapshotinfo__statemachinestorage_.md), we saw how snapshotting helps manage the size of the [RaftLog](05_raftlog_.md) by periodically saving the application's entire state. This is great for keeping logs trim over time.

But what if you need to handle a very large piece of data as part of a *single* operation? Imagine you're building a distributed file storage system on top of Ratis, and a user wants to upload a 1 GB video file. If you tried to put the entire 1 GB video into a single Raft log entry, it would cause several problems:

*   **Log Bloat:** The [RaftLog](05_raftlog_.md) would become enormous very quickly.
*   **Leader Bottleneck:** The Leader [RaftServer](04_raftserver_.md) would have to receive the 1 GB file and then forward it to all Followers, potentially overwhelming its network bandwidth and memory.
*   **Inefficiency:** Replicating 1 GB through the standard Raft log mechanism isn't optimized for bulk data transfer.

How can we efficiently send large amounts of data to the Ratis cluster without choking the regular Raft log replication? This is exactly the problem the **DataStream API** solves.

## What Problem Does the DataStream API Solve?

The DataStream API provides a specialized, alternative path for efficiently streaming large amounts of data directly to the servers in the Ratis cluster.

Think of it like this: Regular Raft log replication (for small commands like "INCREMENT") is like sending a memo through an office mail system. It goes to the central coordinator (the Leader), who then makes copies and distributes them. This works well for small messages.

The DataStream API is like using a **dedicated freight service** for a large package (your 1 GB video file). Instead of going through the regular office mail:

1.  You might send the package directly to the recipient's nearest regional depot (the closest Ratis server).
2.  The freight service ensures copies of the package arrive at *all* necessary regional depots (all Ratis servers in the group), potentially using optimized routes.
3.  Once all depots confirm they have the package, a *small notification memo* (a commit message) is sent through the regular office mail (Raft log) saying "Package #XYZ has arrived and is ready".
4.  When the final recipients (your [StateMachine](01_statemachine_.md) instances) get this memo, they know they can retrieve the large package from their local depot.

## What is the DataStream API?

The DataStream API allows clients to:

*   **Stream Data Directly:** Send large data (like a file) in chunks, possibly to the nearest or most convenient [RaftPeer](02_raftgroup___raftpeer_.md).
*   **Parallel Distribution:** The Ratis servers coordinate to distribute these chunks efficiently among themselves, often in parallel, using a separate data streaming protocol (like Netty).
*   **Out-of-Band Storage:** Servers temporarily store the incoming data chunks outside the main [RaftLog](05_raftlog_.md).
*   **Commit via Raft Log:** Once all data chunks are successfully received by the group, the Leader proposes a *small* message via the standard Raft log. This message doesn't contain the large data itself, but rather a reference or identifier pointing to the data that was just streamed.
*   **StateMachine Access:** When your [StateMachine](01_statemachine_.md) applies this commit message, it uses the identifier to access the large data that was previously streamed and stored locally by the server.

This approach keeps the Raft log small and fast, leveraging a more suitable mechanism for bulk data transfer.

## How to Use the DataStream API (Client-Side)

Let's see how our application (acting as a [RaftClient](03_raftclient_.md)) can use the DataStream API to send a large piece of data.

### 1. Get the DataStream API

First, you get the `DataStreamApi` from your `RaftClient` instance.

```java
import org.apache.ratis.client.RaftClient;
import org.apache.ratis.client.api.DataStreamApi;

// Assume 'client' is your configured RaftClient instance
// RaftClient client = ...;

// Get the DataStream API interface
DataStreamApi dataStreamApi = client.getDataStreamApi();

System.out.println("Obtained DataStream API.");
```

**Explanation:**
This simple step retrieves the specialized API interface for data streaming from the main `RaftClient` object.

### 2. Create a Stream

Next, you start a new stream using the `stream()` method. This returns a `DataStreamOutput` object, which represents your active stream connection.

```java
import org.apache.ratis.client.api.DataStreamOutput;
import java.nio.ByteBuffer;

// Optional: You can provide a header message (e.g., filename, metadata)
// ByteBuffer header = ByteBuffer.wrap("my_large_video.mp4".getBytes());
ByteBuffer header = null; // Or no header

// Create the stream output object
DataStreamOutput streamOutput = dataStreamApi.stream(header);

System.out.println("Created data stream output.");
```

**Explanation:**
Calling `stream()` initiates the process. Behind the scenes, the client might contact a server to set up the stream. You get back a `streamOutput` object that you'll use to write data. You can optionally include a `ByteBuffer` as a header, which might contain metadata about the stream (like a filename).

### 3. Write Data Chunks

Now, you write your large data to the stream in chunks using `writeAsync()`. This method is asynchronous, returning a `CompletableFuture` that completes when the chunk has been processed by the client-side buffer or network layer.

```java
import java.util.concurrent.CompletableFuture;
import java.nio.ByteBuffer;

// Prepare some data chunks (e.g., read from a file)
ByteBuffer chunk1 = ByteBuffer.wrap("This is the first part of the large data.".getBytes());
ByteBuffer chunk2 = ByteBuffer.wrap(" This is the second part.".getBytes());

// Write the chunks asynchronously
CompletableFuture<?> future1 = streamOutput.writeAsync(chunk1);
CompletableFuture<?> future2 = streamOutput.writeAsync(chunk2);
// ... write more chunks ...

// You can wait for writes if needed (e.g., for flow control)
// future1.get(); future2.get();

System.out.println("Sent data chunks asynchronously.");
```

**Explanation:**
You send your data piece by piece using `writeAsync`. This doesn't wait for the data to reach all servers, but rather indicates the request to send has been accepted. The client library handles sending these chunks efficiently to the Ratis cluster peers.

### 4. Close the Stream (Commit)

Once you've written all your data chunks, you **must** close the stream using `closeAsync()`. This is the crucial step that signals to the Ratis cluster: "I'm done sending data for this stream."

Closing the stream triggers the final phase:
*   The Ratis servers verify they have all the data chunks.
*   The Leader proposes the small commit message (with the data reference) via the standard [RaftLog](05_raftlog_.md).
*   Raft consensus occurs for this commit message.

The `CompletableFuture` returned by `closeAsync()` completes *after* the commit message has been successfully replicated and applied via Raft, giving you back a `RaftClientReply`.

```java
import org.apache.ratis.protocol.RaftClientReply;
import java.util.concurrent.CompletableFuture;

// Close the stream and trigger the commit message
CompletableFuture<RaftClientReply> closeFuture = streamOutput.closeAsync();

// Wait for the commit to complete and get the reply
try {
    RaftClientReply reply = closeFuture.get(); // Blocks until commit finishes

    if (reply.isSuccess()) {
        System.out.println("Stream closed and data committed successfully!");
        System.out.println("Commit Log Index: " + reply.getLogIndex());
        // The reply message might contain the data ID assigned by the StateMachine
        // String dataId = reply.getMessage().getContent().toStringUtf8();
    } else {
        System.err.println("Stream commit failed: " + reply.getException());
    }
} catch (Exception e) {
    System.err.println("Error closing stream or waiting for commit: " + e);
}
```

**Explanation:**
Calling `closeAsync()` tells Ratis you've sent all the data. The operation isn't truly complete until the corresponding commit message makes it through the Raft consensus process. The returned `CompletableFuture<RaftClientReply>` allows you to wait for this confirmation. A successful `RaftClientReply` means the cluster has acknowledged receiving the data and logged the commit message.

**Important:** Don't forget to close the `DataStreamApi` itself (which might close underlying connections) when your application is done with it.

```java
// When application shuts down or is done streaming
try {
    dataStreamApi.close();
} catch (IOException e) {
    // Handle close exception
}
```

## How the StateMachine Uses Streamed Data (Server-Side)

Okay, the client streamed the data, and a commit message was logged. How does your [StateMachine](01_statemachine_.md) actually use the large data?

1.  **Implement Data Handling:** Your `StateMachine` needs logic to handle data streams. It might directly implement `org.apache.ratis.statemachine.DataStreamApi` or use information passed via the `TransactionContext`.
2.  **Receive Commit Message:** The `applyTransaction(TransactionContext trx)` method will be called with the small commit message that the Leader proposed after the stream was closed.
3.  **Get Data Reference:** This commit message (or the `TransactionContext`) will contain the identifier/reference for the data that was streamed "out-of-band".
4.  **Access Data:** Ratis provides mechanisms for the StateMachine to access the streamed data using this identifier. The data should be locally available on the server where the StateMachine is running because it was delivered via the DataStream pathway. The StateMachine might read it from a specific file location managed by Ratis or access it via an input stream provided through the transaction context or a related API.
5.  **Process Data:** The StateMachine can now process the large data (e.g., save the video file permanently, parse the large dataset).

Here's a conceptual `applyTransaction` snippet:

```java
// Inside your StateMachine implementing data stream handling

@Override
public CompletableFuture<Message> applyTransaction(TransactionContext trx) {
    LogEntryProto logEntry = trx.getLogEntryUnsafe();

    // Check if this log entry corresponds to a completed data stream
    if (logEntry.hasDataStreamLogEntry()) { // Check for data stream marker
        DataStreamRequestHeaderProto header = logEntry.getDataStreamLogEntry().getHeader();
        long streamId = header.getStreamId();
        long dataLength = header.getDataLength();

        LOG.info("Applying commit for Data Stream ID: {}, Length: {}", streamId, dataLength);

        try {
            // 1. Access the streamed data using the streamId/context
            //    (Mechanism depends on SM implementation - Ratis provides helpers)
            // InputStream dataStream = getDataInputStreamFromRatis(streamId); // Conceptual
            File dataFile = getDataFileFromRatis(streamId); // Conceptual

            // 2. Process the large data (e.g., save to final location)
            processLargeData(dataFile); // Your application logic here

            // 3. Maybe clean up temporary stream data if needed
            cleanupStreamData(streamId); // Conceptual

            // 4. Return success (maybe the data ID or path)
            return CompletableFuture.completedFuture(Message.valueOf("Processed stream " + streamId));

        } catch (IOException e) {
            LOG.error("Failed to process data stream {}", streamId, e);
            // Return failure
            return CompletableFuture.failedFuture(e);
        }

    } else {
        // Handle regular (non-streamed) transactions
        // ...
    }
}

// Conceptual helper methods (implementation details provided by Ratis/SM base classes)
// private InputStream getDataInputStreamFromRatis(long streamId) throws IOException { ... }
// private File getDataFileFromRatis(long streamId) throws IOException { ... }
// private void processLargeData(File dataFile) throws IOException { ... }
// private void cleanupStreamData(long streamId) { ... }
```

**Explanation:**
1.  The `applyTransaction` method receives the commit log entry.
2.  It checks if this entry is specifically for a data stream (e.g., using `hasDataStreamLogEntry()`).
3.  It extracts information like the stream ID.
4.  It uses conceptual helper methods (provided/implemented based on Ratis APIs) to get access to the actual data (as a `File` or `InputStream`) using the ID. Ratis ensures this data is locally present from the streaming phase.
5.  It calls the application-specific logic (`processLargeData`) to handle the large file/data.
6.  It optionally cleans up temporary resources related to the stream.
7.  It returns a success or failure message.

## Internal Implementation Walkthrough

What happens under the hood when you use the DataStream API?

1.  **Initiation (`stream()`):** The `RaftClient` contacts one or more servers (potentially the nearest, based on `RoutingTable`) using the DataStream RPC channel (e.g., Netty) to announce a new stream. Servers prepare to receive data for this stream ID.
2.  **Data Chunking (`writeAsync()`):** The client breaks the data into chunks. For each chunk, it sends it via the DataStream RPC channel. The target server(s) might be chosen based on proximity, or the primary server might forward chunks to others. All servers in the group must eventually receive all chunks.
3.  **Server Reception:** Each server receives data chunks via its `DataStreamServerRpc` endpoint. It stores these chunks temporarily (e.g., in memory buffers or temporary files) associated with the stream ID. Servers communicate among themselves to track which chunks have been received by whom.
4.  **Stream Completion (`closeAsync()`):** The client sends a final message indicating the end of the stream.
5.  **Leader Coordination:** The Leader verifies that all servers in the group have successfully received *all* chunks for the stream.
6.  **StateMachine Linking (Optional but common):** The Leader might interact with its StateMachine (e.g., via a `link` method if the SM implements `DataStreamApi`) to let it know the stream is complete. The SM might move the temporary data to a staging area and assign a persistent identifier.
7.  **Raft Commit:** The Leader constructs a *small* `LogEntryProto` containing the stream identifier (and maybe metadata like total size) and marks it as a DataStream commit entry. It then proposes this entry via the standard [RaftLog](05_raftlog_.md) replication using the normal [RPC Layer](06_rpc_layer__rpctype__rpcfactory__raftserverrpc__raftclientrpc_.md).
8.  **Consensus:** Followers receive, persist, and acknowledge the commit log entry just like any other Raft entry. The commit index advances.
9.  **StateMachine Apply:** All servers (Leader and Followers) eventually deliver the commit log entry to their local `StateMachine` via `applyTransaction`.
10. **Data Access & Processing:** The `StateMachine`, using the identifier from the log entry, accesses the data (which is already locally available from the streaming phase) and performs its application logic.
11. **Client Reply:** Once the commit log entry is applied on the Leader, the Leader sends the final `RaftClientReply` back to the client that called `closeAsync()`.

Here's a simplified sequence diagram:

```mermaid
sequenceDiagram
    participant C as RaftClient
    participant P as Primary/Nearest Server
    participant L as Leader Server
    participant F as Follower Server
    participant SM as StateMachine (on L & F)

    Note over C, F: Client wants to stream large data
    C->>P: Initiate Stream (DataStream RPC)
    P-->>C: Stream ID Assigned
    C->>P: Data Chunk 1 (DataStream RPC)
    P->>L: Forward Chunk 1 (Internal Stream Relay)
    P->>F: Forward Chunk 1 (Internal Stream Relay)
    C->>P: Data Chunk 2 (DataStream RPC)
    P->>L: Forward Chunk 2
    P->>F: Forward Chunk 2
    Note over P,L,F: Servers store chunks temporarily
    C->>P: Close Stream (DataStream RPC)
    Note over P,L,F: Servers confirm all chunks received
    L->>L: Generate DataStream Commit Log Entry (ID=xyz)
    L->>L: Append Commit Entry to RaftLog
    L->>F: AppendEntries(Commit Entry ID=xyz) (Raft RPC)
    F->>F: Append Commit Entry to RaftLog
    F-->>L: Ack Entry
    Note over L: Majority Ack received, Commit Index advances
    L->>SM: applyTransaction(Commit Entry ID=xyz)
    SM->>SM: Access local data for ID=xyz, process it
    SM-->>L: Apply Result
    L-->>C: RaftClientReply(Success, LogIndex=N)
    F->>SM: applyTransaction(Commit Entry ID=xyz) (later)
    SM->>SM: Access local data for ID=xyz, process it
```

## Code Dive (Light)

*   **Client API:**
    *   `org.apache.ratis.client.RaftClient`: Provides `getDataStreamApi()`.
    *   `org.apache.ratis.client.api.DataStreamApi`: Defines the `stream()` method. Found in `ratis-client-api`.
    *   `org.apache.ratis.client.api.DataStreamOutput`: Defines `writeAsync()` and `closeAsync()`. Found in `ratis-client-api`.
    *   `org.apache.ratis.client.DataStreamClientRpc`: Interface for the client-side network component handling the actual streaming protocol. Implementations exist for Netty (`NettyClientStreamRpc`). Found in `ratis-client`.

    ```java
    // Getting the API from RaftClient (simplified from RaftClientImpl)
    // public DataStreamApi getDataStreamApi() {
    //    // Lazily initializes and returns the DataStreamClient object
    //    // which implements DataStreamApi
    //    return dataStreamClient.get();
    // }

    // Example DataStreamOutput usage
    DataStreamApi streamApi = client.getDataStreamApi();
    DataStreamOutput output = streamApi.stream();
    // Asynchronously write data
    CompletableFuture<?> writeFuture = output.writeAsync(ByteBuffer.wrap("data".getBytes()));
    // MUST close to commit
    CompletableFuture<RaftClientReply> commitFuture = output.closeAsync();
    ```

*   **Server API:**
    *   `org.apache.ratis.server.DataStreamServerRpc`: Interface for the server-side network component listening for incoming streams and chunks. Implementations exist for Netty (`NettyServerStreamRpc`). Found in `ratis-server-api`.
    *   `org.apache.ratis.server.DataStreamServerFactory`: Creates the `DataStreamServerRpc` instance, similar to how `ServerFactory` creates `RaftServerRpc`. Found in `ratis-server-api`.
    *   `org.apache.ratis.statemachine.StateMachine`: Your implementation needs logic in `applyTransaction` to handle the commit message and access the streamed data. It might optionally implement `org.apache.ratis.statemachine.DataStreamApi` for closer integration with the stream lifecycle (like the `link` method).

*   **Configuration:** The type of data stream implementation (usually Netty) is configured via `RaftProperties`, similar to the regular RPC layer. See [Chapter 9: Configuration & Properties](09_configuration___properties__raftproperties__raftserverconfigkeys__etc__.md).
    ```java
    import org.apache.ratis.conf.RaftProperties;
    import org.apache.ratis.RaftConfigKeys;
    import org.apache.ratis.datastream.SupportedDataStreamType;

    RaftProperties properties = new RaftProperties();
    // Use Netty for the Data Stream transport
    RaftConfigKeys.DataStream.setType(properties, SupportedDataStreamType.NETTY);

    // Servers and clients need the corresponding datastream port configured too
    // RaftConfigKeys.DataStream.setPort(properties, dataStreamPort);
    ```

## Conclusion

The DataStream API is a powerful feature in Ratis designed specifically for handling large data efficiently. By separating the bulk data transfer onto an optimized streaming path and using the standard Raft log only for a small commit message, it avoids bloating the log and improves throughput for data-intensive applications. The client uses `DataStreamApi` and `DataStreamOutput` to stream data, and the `StateMachine` accesses the locally stored data when processing the corresponding commit message in `applyTransaction`.

Key Takeaways:

*   DataStream API is for efficiently handling large data transfers.
*   It streams data chunks directly to servers, often bypassing the Leader initially.
*   Data is stored temporarily outside the Raft log.
*   A small commit message with a data reference is replicated via the standard Raft log.
*   The StateMachine uses the reference in `applyTransaction` to access the locally streamed data.
*   It uses a separate, pluggable network layer (usually Netty).

We've now seen how Ratis handles core consensus, logging, networking, snapshots, and even large data streams. But how do we fine-tune all these components? The next chapter delves into the various configuration options available in Ratis.

**Next:** [Chapter 9: Configuration & Properties (RaftProperties, RaftServerConfigKeys, etc.)](09_configuration___properties__raftproperties__raftserverconfigkeys__etc__.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)