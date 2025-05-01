# Chapter 9: Configuration & Properties (RaftProperties, RaftServerConfigKeys, etc.)

Welcome to Chapter 9! In the [previous chapter](08_datastream_api_.md), we learned how Ratis uses the DataStream API to efficiently handle very large pieces of data, separating bulk transfer from the regular Raft log.

So far, we've seen how to define our application logic ([StateMachine](01_statemachine_.md)), set up the cluster ([RaftGroup & RaftPeer](02_raftgroup___raftpeer_.md)), interact with it ([RaftClient](03_raftclient_.md)), run the core server process ([RaftServer](04_raftserver_.md)), manage the log ([RaftLog](05_raftlog_.md)), handle networking ([RPC Layer](06_rpc_layer__rpctype__rpcfactory__raftserverrpc__raftclientrpc_.md)), compact the log ([Snapshotting](07_snapshotting__snapshotinfo__statemachinestorage_.md)), and stream data ([DataStream API](08_datastream_api_.md)). But how do we control all the different knobs and switches for these components? What if the default settings aren't quite right for our specific needs?

## What Problem Does Configuration Solve?

Imagine you're setting up your Ratis-based counter service. By default, Ratis might store its log files in a temporary directory like `/tmp/raft-server/`. That's fine for a quick test, but for a real application, you definitely want to store the data somewhere more permanent and reliable, like `/var/lib/my-counter-service/`.

Or perhaps you notice that in your network environment, the default timeouts for leader election (how long a follower waits before trying to become leader) are a bit too short, causing unnecessary elections. You want to make them slightly longer.

How do you tell Ratis to use a different storage directory? How do you adjust those timeout values? How do you switch between different networking libraries like gRPC or Netty for the [RPC Layer](06_rpc_layer__rpctype__rpcfactory__raftserverrpc__raftclientrpc_.md)?

This is where **Configuration & Properties** come in. They provide the mechanism to customize and tune the behavior of Ratis clients and servers. Think of it as the **settings menu** or the **operating manual** for your Ratis cluster.

## Key Concepts

### 1. `RaftProperties`: The Settings Container

The central class for holding configuration is `org.apache.ratis.conf.RaftProperties`. It's essentially a map that stores configuration settings as key-value pairs, where both the key and the value are typically strings.

It's similar to Java's built-in `java.util.Properties` class but tailored for Ratis. You create an instance of `RaftProperties` and then load it with the settings you want to change from their defaults.

```java
import org.apache.ratis.conf.RaftProperties;

// Create an empty properties object
RaftProperties properties = new RaftProperties();

// We can add settings to this object
```

### 2. Configuration Keys Classes (e.g., `RaftServerConfigKeys`)

How do you know *which* keys to put into `RaftProperties`? What are the valid setting names, and what do they control?

Ratis provides several "ConfigKeys" classes that define the standard configuration parameters, their string keys, their default values, and often helper methods to get and set them in a type-safe way.

Some important examples include:

*   **`org.apache.ratis.server.RaftServerConfigKeys`**: Defines settings specific to the [RaftServer](04_raftserver_.md), such as:
    *   Storage directory (`raft.server.storage.dir`)
    *   Election timeouts (`raft.server.rpc.timeout.min`, `raft.server.rpc.timeout.max`)
    *   Snapshot triggers (`raft.server.snapshot.auto.trigger.threshold`)
    *   Log segment sizes (`raft.server.log.segment.size.max`)
    *   Enabling/disabling features like pre-vote (`raft.server.leaderelection.pre-vote`)
*   **`org.apache.ratis.client.RaftClientConfigKeys`**: Defines settings for the [RaftClient](03_raftclient_.md), such as:
    *   RPC request timeouts (`raft.client.rpc.request.timeout`)
    *   Limits for asynchronous requests (`raft.client.async.outstanding-requests.max`)
*   **`org.apache.ratis.grpc.GrpcConfigKeys`**: Defines settings specific to using gRPC for the [RPC Layer](06_rpc_layer__rpctype__rpcfactory__raftserverrpc__raftclientrpc_.md), like network ports.
*   **`org.apache.ratis.netty.NettyConfigKeys`**: Defines settings for using Netty for RPC or DataStream.
*   **`org.apache.ratis.RaftConfigKeys`**: Defines some common keys applicable to both client and server, like the RPC type or DataStream type.

These classes act as a **dictionary** for Ratis settings. They help you avoid typos in keys and make it clear what options are available.

### 3. Pluggability via Properties

Configuration is also how Ratis achieves pluggability for components like the [RPC Layer](06_rpc_layer__rpctype__rpcfactory__raftserverrpc__raftclientrpc_.md) or the [RaftLog](05_raftlog_.md). You set a property to tell Ratis *which implementation* you want to use.

```java
import org.apache.ratis.conf.RaftProperties;
import org.apache.ratis.RaftConfigKeys;
import org.apache.ratis.rpc.SupportedRpcType;

RaftProperties properties = new RaftProperties();

// Tell Ratis to use the Netty implementation for RPC
RaftConfigKeys.Rpc.setType(properties, SupportedRpcType.NETTY);
```

When Ratis builds the client or server, it reads this property and loads the appropriate factory and implementation (e.g., `NettyFactory`, `NettyServerRpc`).

## How to Use Configuration

Let's configure our counter server to use a specific storage directory and adjust the minimum election timeout.

### 1. Create `RaftProperties`

Start by creating an empty `RaftProperties` object.

```java
import org.apache.ratis.conf.RaftProperties;

RaftProperties serverProperties = new RaftProperties();
```

### 2. Set Properties using `*ConfigKeys` Helpers

Use the static methods provided in the `*ConfigKeys` classes. This is the **recommended** way as it's type-safe and less prone to errors.

```java
import org.apache.ratis.server.RaftServerConfigKeys;
import org.apache.ratis.util.TimeDuration;
import java.io.File;
import java.util.Collections;
import java.util.concurrent.TimeUnit;

// Set the storage directory
List<File> storageDirs = Collections.singletonList(new File("/var/lib/my-counter-service"));
RaftServerConfigKeys.setStorageDir(serverProperties, storageDirs);

// Set the minimum election timeout to 500 milliseconds
TimeDuration minTimeout = TimeDuration.valueOf(500, TimeUnit.MILLISECONDS);
RaftServerConfigKeys.Rpc.setTimeoutMin(serverProperties, minTimeout);

System.out.println("Storage Dir Set To: " + RaftServerConfigKeys.storageDir(serverProperties));
System.out.println("Min Timeout Set To: " + RaftServerConfigKeys.Rpc.timeoutMin(serverProperties));
```

**Explanation:**

*   We use `RaftServerConfigKeys.setStorageDir()` to set the `raft.server.storage.dir` property.
*   We use `RaftServerConfigKeys.Rpc.setTimeoutMin()` to set the `raft.server.rpc.timeout.min` property.
*   The helper methods often take the correct type (like `List<File>` or `TimeDuration`), converting it to a string internally when storing it in `RaftProperties`.
*   Matching getter methods (`storageDir()`, `timeoutMin()`) allow you to read the values back, applying defaults if not set.

### 3. Set Properties using Raw Strings (Alternative)

You can also set properties using their raw string keys. This is less safe as typos won't be caught at compile time.

```java
// Setting the maximum election timeout using the raw key
serverProperties.set("raft.server.rpc.timeout.max", "1000ms"); // 1000 milliseconds

// Reading it back using the helper still works
TimeDuration maxTimeout = RaftServerConfigKeys.Rpc.timeoutMax(serverProperties);
System.out.println("Max Timeout Set To (Raw): " + maxTimeout);
```

**Explanation:**
We use the standard `set(key, value)` method of `RaftProperties`. Note that we need to provide the value as a string in the format Ratis expects (e.g., "1000ms" for `TimeDuration`).

### 4. Apply Properties to Client/Server

Finally, pass the configured `RaftProperties` object to the builder when creating your `RaftClient` or `RaftServer`.

```java
// Assume 'thisPeerId', 'counterGroup', 'counterStateMachine' are defined

// Build the RaftServer using the configured properties
RaftServer server = RaftServer.newBuilder()
        .setServerId(thisPeerId)
        .setGroup(counterGroup)
        .setProperties(serverProperties) // Apply our custom settings!
        .setStateMachine(counterStateMachine)
        // .setOption(...)
        .build();

// When server.start() is called, it will use the settings from serverProperties
// (e.g., storing logs in /var/lib/my-counter-service)
```

The builder reads the properties and configures the server's components accordingly. The same pattern applies to `RaftClient.newBuilder()`.

## How Configuration Works Internally

When you call `.build()` on a `RaftServer` or `RaftClient` builder:

1.  **Properties Stored:** The builder stores the provided `RaftProperties` object.
2.  **Component Creation:** As the builder constructs various internal components (like the [RPC Layer](06_rpc_layer__rpctype__rpcfactory__raftserverrpc__raftclientrpc_.md) or [RaftLog](05_raftlog_.md)), it passes the `RaftProperties` to them or their factories.
3.  **Reading Settings:** Each component (or its factory) uses the appropriate `*ConfigKeys` helper methods (or raw `get` calls) to read the settings it needs from the `RaftProperties` object. It uses default values if a specific property isn't found.
    *   Example: The `SegmentedRaftLog` constructor reads `RaftServerConfigKeys.Log.segmentSizeMax(properties)` to determine the maximum size for log segment files.
    *   Example: The `RaftServerProxy` (which manages RPC) reads `RaftConfigKeys.Rpc.type(properties)` to find out whether to create a `GrpcFactory` or `NettyFactory`.
4.  **Component Behavior:** The components then use these retrieved values to configure their behavior (e.g., setting buffer sizes, timeouts, storage locations).

```mermaid
graph TD
    A[MyApp Creates RaftProperties] --> B{Set Properties using ConfigKeys Helpers};
    B --> C[Pass Properties to RaftServer.Builder];
    C --> D[Builder.build()];
    D --> E{Server Creates RaftLog};
    D --> F{Server Creates RPC Layer};
    D --> G{Server Creates Other Components...};
    E --> H[RaftLog Reads Log Settings from Properties];
    F --> I[RPC Layer Reads RPC Type/Port from Properties];
    G --> J[Components Read Their Settings];

    subgraph RaftServer Internals
        E; F; G; H; I; J;
    end
```

### `Parameters` Object

While `RaftProperties` handles string-based key-value configuration, sometimes Ratis needs to pass around more complex, non-string configuration objects during initialization (like a pre-configured TLS context). For this, it uses a simple class called `org.apache.ratis.conf.Parameters`. This acts as a type-safe map for arbitrary objects, often used internally between factories and components, especially for things that aren't easily represented as strings. You typically won't interact with `Parameters` directly unless you're implementing custom factories or deeply customizing Ratis internals.

## Code Dive

### `RaftProperties`

The `org.apache.ratis.conf.RaftProperties` class is relatively straightforward. It primarily wraps a `ConcurrentMap<String, String>`.

```java
// From: ratis-common/src/main/java/org/apache/ratis/conf/RaftProperties.java
public class RaftProperties {
  // Internal map to store properties
  private final ConcurrentMap<String, String> properties = new ConcurrentHashMap<>();

  // Constructor
  public RaftProperties() { }

  // Get a value (handles variable substitution)
  public String get(String name) {
    return substituteVars(getRaw(name));
  }

  // Get raw value without substitution
  public String getRaw(String name) {
    return properties.get(Objects.requireNonNull(name, "name == null").trim());
  }

  // Set a value
  public void set(String name, String value) {
    final String trimmed = Objects.requireNonNull(name, "name == null").trim();
    Objects.requireNonNull(value, () -> "value == null for " + trimmed);
    properties.put(trimmed, value);
  }

  // Helper methods for specific types (use ConfigKeys classes instead where possible)
  public Integer getInt(String name, Integer defaultValue) { /* ... */ }
  public void setInt(String name, int value) { /* ... */ }
  public TimeDuration getTimeDuration(String name, TimeDuration defaultValue, TimeUnit unit) { /* ... */ }
  public void setTimeDuration(String name, TimeDuration value) { /* ... */ }
  // ... other helpers for boolean, long, double, SizeInBytes, Class, File(s) ...
}
```

### `RaftServerConfigKeys` Example

Let's look at the pattern used in `RaftServerConfigKeys`.

```java
// From: ratis-server-api/src/main/java/org/apache/ratis/server/RaftServerConfigKeys.java
public interface RaftServerConfigKeys {
  String PREFIX = "raft.server";

  // --- Storage Directory Example ---
  String STORAGE_DIR_KEY = PREFIX + ".storage.dir"; // The actual key string
  // Default value if the key is not set
  List<File> STORAGE_DIR_DEFAULT = Collections.singletonList(new File("/tmp/raft-server/"));

  // Getter method: Reads from properties, applies default
  static List<File> storageDir(RaftProperties properties) {
    // Uses ConfUtils.getFiles helper internally
    return getFiles(properties::getFiles, STORAGE_DIR_KEY, STORAGE_DIR_DEFAULT, getDefaultLog());
  }

  // Setter method: Writes to properties
  static void setStorageDir(RaftProperties properties, List<File> storageDir) {
    // Uses ConfUtils.setFiles helper internally
    setFiles(properties::setFiles, STORAGE_DIR_KEY, storageDir);
  }

  // --- Min RPC Timeout Example ---
  interface Rpc { // Nested interface for related keys
    String PREFIX = RaftServerConfigKeys.PREFIX + ".rpc";
    String TIMEOUT_MIN_KEY = PREFIX + ".timeout.min";
    TimeDuration TIMEOUT_MIN_DEFAULT = TimeDuration.valueOf(150, TimeUnit.MILLISECONDS);

    static TimeDuration timeoutMin(RaftProperties properties) {
       return getTimeDuration(properties.getTimeDuration(TIMEOUT_MIN_DEFAULT.getUnit()),
          TIMEOUT_MIN_KEY, TIMEOUT_MIN_DEFAULT, /*...*/);
    }
    static void setTimeoutMin(RaftProperties properties, TimeDuration minDuration) {
       setTimeDuration(properties::setTimeDuration, TIMEOUT_MIN_KEY, minDuration);
    }
    // ... other RPC keys ...
  }
  // ... many other keys and nested interfaces ...
}
```

**Explanation:**

*   Each setting has a `_KEY` constant defining the string identifier (e.g., `raft.server.storage.dir`).
*   A `_DEFAULT` constant defines the fallback value.
*   A static getter method (e.g., `storageDir(properties)`) provides a convenient way to read the value, handling defaults and type conversion. It often uses helper methods from `org.apache.ratis.conf.ConfUtils`.
*   A static setter method (e.g., `setStorageDir(properties, value)`) provides a type-safe way to set the value.
*   Related keys are often grouped within nested interfaces (like `Rpc`, `Log`, `Snapshot`).

### Ratis Configuration Reference

For a comprehensive list of all available configuration keys and their defaults, you can refer to the Ratis documentation:
*   [Ratis Configuration Reference](https://ratis.apache.org/docs/configurations.html)
    (Or refer to the `ratis-docs/src/site/markdown/configurations.md` file provided in the context).

This documentation is generated directly from the `*ConfigKeys` classes and provides the most up-to-date reference.

## Conclusion

Configuration is essential for adapting Ratis to your specific environment and application needs. The `RaftProperties` object holds your custom settings as key-value pairs. You should use the helper methods in the various `*ConfigKeys` classes (like `RaftServerConfigKeys`, `RaftClientConfigKeys`, `GrpcConfigKeys`) to set these properties in a type-safe manner. By passing the configured `RaftProperties` to the client or server builders, you can control everything from storage locations and timeouts to network implementations and feature flags.

Key Takeaways:

*   Configuration allows tuning Ratis behavior.
*   `RaftProperties` holds key-value settings.
*   `*ConfigKeys` classes (e.g., `RaftServerConfigKeys`) define standard keys, defaults, and provide helper get/set methods.
*   Use helper methods for type safety.
*   Pass `RaftProperties` to builders (`RaftClient.newBuilder()`, `RaftServer.newBuilder()`) to apply settings.
*   Properties control internal component behavior and pluggability.

Now that we know how to configure Ratis, how can we observe its internal state and performance while it's running? The final chapter explores how Ratis exposes metrics.

**Next:** [Chapter 10: Metrics (RatisMetricRegistry, MetricRegistries)](10_metrics__ratismetricregistry__metricregistries__.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)