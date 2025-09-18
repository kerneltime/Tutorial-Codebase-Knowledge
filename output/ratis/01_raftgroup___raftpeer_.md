# Chapter 1: RaftGroup & RaftPeer

Welcome to your first step into the world of Apache Ratis! If you're looking to build systems that are reliable and can continue working even if some parts fail, you're in the right place. Ratis helps you do this using something called the Raft consensus algorithm.

But before we dive into complex algorithms, let's start with the basics: how do different parts of our distributed system identify each other and know who they're supposed to be working with?

Imagine you're building a super-reliable online service, say, a shared digital whiteboard where multiple users can draw simultaneously, and everyone sees the same picture. To make this reliable, several computers (servers) will work together. They need to agree on every line drawn.
*   How does Server A know it should talk to Server B and Server C for this whiteboard application?
*   How do they uniquely identify each other?
*   What if these same servers are *also* working on a different task, like a shared document editor? How do they keep these collaborations separate?

This is where `RaftGroup` and `RaftPeer` come into play. They are like the address book and team roster for our distributed system.

## What is a RaftPeer? Your Server's Contact Card

Think of a `RaftPeer` as the contact card for an individual server participating in the consensus process. It tells you *who* the server is and *how to reach it*.

But before we have a full contact card (`RaftPeer`), we need a unique name or identifier for each server.

### `RaftPeerId`: The Unique Name Tag

Every server in a Ratis cluster that participates in the consensus needs a unique identifier. This is its `RaftPeerId`. It's like a unique username or a serial number for that server instance within the context of Ratis.

You can create a `RaftPeerId` simply from a string:

```java
import org.apache.ratis.protocol.RaftPeerId;

// Create a RaftPeerId from a string
RaftPeerId serverOneId = RaftPeerId.valueOf("server1");
RaftPeerId serverTwoId = RaftPeerId.valueOf("s2"); // Can be short too!

System.out.println("Server One ID: " + serverOneId);
System.out.println("Server Two ID: " + serverTwoId);
```

If you run this, the output would be:

```
Server One ID: server1
Server Two ID: s2
```

This `RaftPeerId` is crucial because it's how servers distinguish each other.

### `RaftPeer`: The Full Contact Information

Now that a server has an ID, we need its "contact card" – the `RaftPeer` object. A `RaftPeer` holds:

1.  The `RaftPeerId` (who it is).
2.  Its network address (e.g., `"localhost:6000"` or `"192.168.1.10:9876"`), which other servers in the same consensus group use to communicate with it.
3.  Optionally, other specific network addresses for different types of communication (like client requests, administrative commands, or high-speed data streaming).
4.  A priority level (which can influence leader election, but we'll keep it simple for now and use defaults).
5.  A startup role, defining its initial state (usually `FOLLOWER` or `LISTENER`).

Let's create a `RaftPeer`:

```java
import org.apache.ratis.protocol.RaftPeer;
import org.apache.ratis.protocol.RaftPeerId;

// First, get a RaftPeerId
RaftPeerId peerId = RaftPeerId.valueOf("nodeA");

// Now, create the RaftPeer using a builder
RaftPeer peerNodeA = RaftPeer.newBuilder()
                             .setId(peerId)
                             .setAddress("127.0.0.1:9000") // Main address for server-to-server talk
                             .setClientAddress("127.0.0.1:9001") // Optional: address for clients
                             .build();

System.out.println("Peer Node A ID: " + peerNodeA.getId());
System.out.println("Peer Node A Address: " + peerNodeA.getAddress());
System.out.println("Peer Node A Client Address: " + peerNodeA.getClientAddress());
System.out.println("Peer Node A Details: " + peerNodeA.getDetails());
```

Running this would output something like:

```
Peer Node A ID: nodeA
Peer Node A Address: 127.0.0.1:9000
Peer Node A Client Address: 127.0.0.1:9001
Peer Node A Details: nodeA|127.0.0.1:9000|client:127.0.0.1:9001|priority:0|startupRole:FOLLOWER
```
*(The exact detail string might vary slightly with Ratis versions if defaults change, but the core info will be there.)*

So, `RaftPeer` bundles the identity (`RaftPeerId`) with the means to communicate (`address`).

## What is a RaftGroup? The Team Roster

Okay, we know how to identify individual servers and their contact info. But Ratis is about *groups* of servers working together to achieve consensus. This "team" is called a `RaftGroup`.

Think of it like a sports team roster: it has a team name and a list of all the players on that team.

### `RaftGroupId`: The Team's Unique Name

Just as individual servers have a `RaftPeerId`, a specific team or group of servers also needs a unique identifier. This is the `RaftGroupId`. This helps distinguish one consensus team from another, especially if the same physical servers are part of multiple teams (more on that later!).

A `RaftGroupId` is typically a UUID (Universally Unique Identifier) to ensure it's globally unique and avoids clashes.

```java
import org.apache.ratis.protocol.RaftGroupId;
import java.util.UUID;

// Create a RaftGroupId using a randomly generated UUID
RaftGroupId whiteboardAppGroupId = RaftGroupId.randomId();

// Or, if you have a specific UUID you want to use:
UUID specificUUID = UUID.fromString("00112233-4455-6677-8899-aabbccddeeff");
RaftGroupId specificEditorGroupId = RaftGroupId.valueOf(specificUUID);

System.out.println("Whiteboard App Group ID: " + whiteboardAppGroupId);
System.out.println("Specific Editor Group ID: " + specificEditorGroupId);
```

Output:

```
Whiteboard App Group ID: group-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Specific Editor Group ID: group-00112233-4455-6677-8899-aabbccddeeff
```
(The `x`'s will be random hexadecimal characters for the first ID).

### `RaftGroup`: The Actual Team Roster

A `RaftGroup` object defines a specific consensus instance. It brings together:

1.  The `RaftGroupId` (the team's unique name).
2.  A list of all the `RaftPeer`s that are members of this specific team.

This tells Ratis precisely which servers are supposed to collaborate for this particular group's consensus.

Let's form a team!

```java
import org.apache.ratis.protocol.RaftGroup;
import org.apache.ratis.protocol.RaftGroupId;
import org.apache.ratis.protocol.RaftPeer;
import org.apache.ratis.protocol.RaftPeerId;
import java.util.Arrays; // For creating a list easily

// 1. Create Peer IDs and Peers for our team members
RaftPeer peer1 = RaftPeer.newBuilder().setId("s1").setAddress("localhost:1001").build();
RaftPeer peer2 = RaftPeer.newBuilder().setId("s2").setAddress("localhost:1002").build();
RaftPeer peer3 = RaftPeer.newBuilder().setId("s3").setAddress("localhost:1003").build();

// 2. Create a Group ID for our team
RaftGroupId ourTeamId = RaftGroupId.randomId();

// 3. Create the RaftGroup (the team roster)
RaftGroup ourAwesomeTeam = RaftGroup.valueOf(ourTeamId, Arrays.asList(peer1, peer2, peer3));

System.out.println("Our Team's ID: " + ourAwesomeTeam.getGroupId());
System.out.println("Peers in our team: ");
for (RaftPeer member : ourAwesomeTeam.getPeers()) {
    System.out.println("  - " + member.getId() + " at " + member.getAddress());
}
System.out.println("Full Team Info: " + ourAwesomeTeam.toString());
```

Running this code will produce output similar to:

```
Our Team's ID: group-yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy
Peers in our team:
  - s1 at localhost:1001
  - s2 at localhost:1002
  - s3 at localhost:1003
Full Team Info: group-yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy:[s1|localhost:1001, s2|localhost:1002, s3|localhost:1003]
```
(The `y`'s will be random characters from the `RaftGroupId`).

Now, `ourAwesomeTeam` defines a specific set of three peers (`s1`, `s2`, `s3`) that will work together under the unique group identity `ourTeamId`.

## One Server, Multiple Teams (Groups)

A very powerful feature of Ratis is that a single Ratis server process (one running Java program) can be a member of *multiple* `RaftGroup`s simultaneously.

Think of a talented person who is part of:
*   The "Book Club" (`RaftGroup` 1) - discussing books.
*   The "Hiking Club" (`RaftGroup` 2) - planning hikes.

This person (the server process) participates in consensus for *both* groups, but the topics (data being agreed upon) and potentially the other members are different for each group.

This allows you to manage different distributed applications or datasets on the same underlying Ratis infrastructure, each with its own independent consensus. For example, one `RaftGroup` might manage user session data, while another `RaftGroup` on the same servers manages a distributed task queue.

## Visualizing RaftGroup and RaftPeers

Let's try to picture this with a diagram:

```mermaid
graph TD
    subgraph RaftSystem
        direction LR

        subgraph Group_Alpha ["RaftGroup A (ID: group-abc-123)"]
            direction TB
            P_A1[Peer 1 (ID: p1, Addr: serverX:8001)]
            P_A2[Peer 2 (ID: p2, Addr: serverY:8001)]
            P_A3[Peer 3 (ID: p3, Addr: serverZ:8001)]
        end

        subgraph Group_Beta ["RaftGroup B (ID: group-xyz-789)"]
            direction TB
            P_B1[Peer 4 (ID: p4, Addr: serverX:9001)]
            P_B2[Peer 5 (ID: p5, Addr: serverM:9001)]
            P_B3[Peer 6 (ID: p6, Addr: serverY:9001)]
        end

        %% Physical Servers
        ServerX["Physical Server X"]
        ServerY["Physical Server Y"]
        ServerZ["Physical Server Z"]
        ServerM["Physical Server M"]

        %% Mapping logical peers to physical servers
        P_A1 --> ServerX
        P_A2 --> ServerY
        P_A3 --> ServerZ

        P_B1 --> ServerX %% ServerX hosts peers for both Group A and Group B
        P_B2 --> ServerM
        P_B3 --> ServerY %% ServerY hosts peers for both Group A and Group B

    end
    
    style Group_Alpha fill:#ddeeff,stroke:#333,stroke-width:2px
    style Group_Beta fill:#ddffdd,stroke:#333,stroke-width:2px
```

In this diagram:
*   We have two distinct `RaftGroup`s: "Group A" and "Group B", each with its own unique `RaftGroupId`.
*   "Group A" has three peers: `P_A1`, `P_A2`, `P_A3`.
*   "Group B" also has three peers: `P_B1`, `P_B2`, `P_B3`.
*   Notice that "Physical Server X" hosts `P_A1` (for Group A) and `P_B1` (for Group B). Similarly, "Physical Server Y" hosts `P_A2` (for Group A) and `P_B3` (for Group B). This illustrates how a single server process can participate in multiple groups, each listening on different addresses or ports for its group-specific communication.

## Under the Hood: A Glimpse at the Code Structure

Let's briefly look at how these concepts are represented in the Ratis codebase. You don't need to memorize this, but it helps to see the connection.

*   **`RaftPeerId.java`** (`ratis-common/src/main/java/org/apache/ratis/protocol/RaftPeerId.java`)
    This class is a wrapper for the unique ID of a peer. It's often a string.

    ```java
    // Simplified snippet from RaftPeerId.java
    public final class RaftPeerId {
        private final String idString; // The ID as a human-readable string
        // Internally, it also stores it as ByteString for efficiency

        private RaftPeerId(String id) {
            this.idString = Objects.requireNonNull(id, "id == null");
            // ...
        }

        public static RaftPeerId valueOf(String id) {
            // Manages creation, often caching instances for common IDs
            return STRING_MAP.computeIfAbsent(id, RaftPeerId::new);
        }

        @Override
        public String toString() {
            return idString;
        }
        // ... other methods like equals(), hashCode() ...
    }
    ```
    This shows `RaftPeerId` mainly encapsulates a string ID. The `valueOf` static factory method is the standard way to get an instance.

*   **`RaftPeer.java`** (`ratis-common/src/main/java/org/apache/ratis/protocol/RaftPeer.java`)
    This class holds the `RaftPeerId`, network addresses, priority, etc. It uses a `Builder` pattern for construction, which makes creating `RaftPeer` objects cleaner, especially when there are multiple optional fields. `RaftPeer` objects are immutable, meaning their state cannot be changed after creation.

    ```java
    // Simplified snippet from RaftPeer.java
    public final class RaftPeer {
        private final RaftPeerId id;
        private final String address; // For server-server communication
        private final String clientAddress; // For client communication
        // ... other addresses like adminAddress, dataStreamAddress ...
        private final int priority;
        private final RaftPeerRole startupRole;

        // Private constructor, used by the Builder
        private RaftPeer(RaftPeerId id, String address, String clientAddress, /*...other params...*/) {
            this.id = id;
            this.address = address;
            this.clientAddress = clientAddress;
            // ...
        }

        public static class Builder {
            private RaftPeerId id;
            private String address;
            private String clientAddress;
            // ...
            public Builder setId(RaftPeerId id) { this.id = id; return this; }
            public Builder setAddress(String address) { this.address = address; return this; }
            public Builder setClientAddress(String clientAddress) { /* ... */ return this; }
            // ...
            public RaftPeer build() {
                return new RaftPeer(id, address, clientAddress, /*...*/);
            }
        }
        // ... getters for id, address, etc. ...
    }
    ```
    The `Builder` inner class provides a fluent way to set the properties of a `RaftPeer` before `build()`-ing the immutable object.

*   **`RaftGroupId.java`** (`ratis-common/src/main/java/org/apache/ratis/protocol/RaftGroupId.java`)
    This class is a specialized ID for a group, typically wrapping a `java.util.UUID`.

    ```java
    // Simplified snippet from RaftGroupId.java
    public final class RaftGroupId extends RaftId { // RaftId is a base for UUID-based IDs
        private RaftGroupId(UUID uuid) {
            super(uuid); // The UUID is stored in the parent RaftId class
        }

        public static RaftGroupId randomId() {
            // Uses an internal factory to create an ID with a new random UUID
            return FACTORY.randomId();
        }

        public static RaftGroupId valueOf(UUID uuid) {
            // Uses an internal factory to get/create an ID for a given UUID
            return FACTORY.valueOf(uuid);
        }
        // ...
    }
    ```
    It ensures group IDs are unique, usually leveraging Java's `UUID`.

*   **`RaftGroup.java`** (`ratis-common/src/main/java/org/apache/ratis/protocol/RaftGroup.java`)
    This class combines a `RaftGroupId` with a collection of `RaftPeer`s. It typically stores the peers in a `Map` for efficient lookup by `RaftPeerId`. Like `RaftPeer`, `RaftGroup` objects are immutable.

    ```java
    // Simplified snippet from RaftGroup.java
    public final class RaftGroup {
        private final RaftGroupId groupId;
        private final Map<RaftPeerId, RaftPeer> peers; // Keyed by RaftPeerId

        private RaftGroup(RaftGroupId groupId, Iterable<RaftPeer> peerCollection) {
            this.groupId = Objects.requireNonNull(groupId, "groupId == null");
            
            final Map<RaftPeerId, RaftPeer> map = new HashMap<>();
            if (peerCollection != null) {
                peerCollection.forEach(p -> map.put(p.getId(), p));
            }
            this.peers = Collections.unmodifiableMap(map); // Makes the map immutable
        }

        public static RaftGroup valueOf(RaftGroupId groupId, RaftPeer... peers) {
            // Helper to create a group from varargs of peers
            return new RaftGroup(groupId, Arrays.asList(peers));
        }
         public static RaftGroup valueOf(RaftGroupId groupId, Iterable<RaftPeer> peers) {
            return new RaftGroup(groupId, peers);
        }

        public RaftGroupId getGroupId() { return groupId; }
        public Collection<RaftPeer> getPeers() { return peers.values(); }
        public RaftPeer getPeer(RaftPeerId id) { return peers.get(id); }
        // ...
    }
    ```
    The `RaftGroup` holds the definitive list of peers for that specific consensus instance.

You might also encounter `RaftConfiguration` (`ratis-server-api/.../RaftConfiguration.java`). While `RaftGroup` defines the *initial* or *known* set of peers, the [Configuration Management](07_configuration_management_.md) (covered much later) deals with how the active membership of a group can change over time (e.g., adding or removing servers). `RaftConfiguration` uses `RaftPeer` and `RaftPeerId` to describe these dynamic memberships. For now, `RaftGroup` is the key concept for understanding the basic team composition.

## Conclusion

Congratulations! You've just learned about the foundational concepts of `RaftPeerId`, `RaftPeer`, `RaftGroupId`, and `RaftGroup` in Ratis.

*   **`RaftPeerId`**: A server's unique name.
*   **`RaftPeer`**: A server's "contact card" (ID, address, etc.).
*   **`RaftGroupId`**: A consensus team's unique name.
*   **`RaftGroup`**: The "team roster" itself, listing all `RaftPeer`s for a specific consensus instance.

These components are essential for defining *who* participates in a distributed consensus process and *how* they are identified and reached. They provide the basic addressing and membership information needed for servers to collaborate.

Now that we understand how to define the "who" (the members and groups), we're ready to explore the "what" and "how." In the next chapter, we'll dive into the [RaftServer](02_raftserver_.md), which is the actual engine that runs the Raft consensus protocol for a `RaftGroup`.

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)