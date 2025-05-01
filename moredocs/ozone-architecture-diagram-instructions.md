# Apache Ozone Architecture Diagram Instructions

## Diagram Overview

The Apache Ozone architecture diagram should be a comprehensive visual representation of the system components and their interactions. Below is a detailed description of what should be included in the diagram:

## Components to Include

1. **Client Layer** (Top)
   - S3 Client
   - Hadoop Compatible Client (ofs)
   - Native Ozone Client
   - CLI Client

2. **Protocol Layer** (Below Clients)
   - S3 Gateway
   - Protocol Bus
   - Native RPC

3. **Metadata Management Layer** (Center)
   - **Ozone Manager (OM)** - Show with:
     - RocksDB Database
     - Ratis Replication (for HA)
     - Key Management
   
   - **Storage Container Manager (SCM)** - Show with:
     - RocksDB Database
     - Pipeline Management
     - Container Management
     - Node Management
     - Ratis Replication (for HA)

4. **Storage Layer** (Bottom)
   - **Multiple Datanodes** - Each with:
     - Container Management
     - RocksDB for metadata
     - Disk Storage
     - Ratis for replication

5. **Management Layer** (Side)
   - **Recon** - Connected to:
     - OM
     - SCM
     - Datanodes
     - Show databases and UI

## Data Flow Arrows

1. **Write Path** - Numbered arrows (1-5) showing:
   1. Client requests OM
   2. OM requests block allocation from SCM
   3. SCM creates pipeline and returns to OM
   4. OM returns pipeline info to client
   5. Client writes directly to datanodes

2. **Read Path** - Different colored arrows showing:
   1. Client requests block info from OM
   2. OM returns block locations
   3. Client reads directly from datanodes

## Visual Style Guidelines

- Use color coding to differentiate components:
  - Client components (light blue)
  - OM components (light green)
  - SCM components (light orange)
  - Datanode components (light purple)
  - Recon components (light yellow)

- Use different line styles for different types of connections:
  - Solid lines for direct communication
  - Dashed lines for replication traffic
  - Dotted lines for monitoring/metrics

- Add a legend explaining the colors and line styles

- Group related components with container boxes

## Layout

- Organize in layers from top to bottom:
  - Client layer
  - Protocol layer
  - Metadata management layer (OM and SCM)
  - Storage layer (Datanodes)

- Place Recon on the right side, connected to all major components

## Creating the Diagram

1. Use a tool like draw.io, Lucidchart, or similar diagramming software
2. Start with the main components as boxes
3. Arrange them in the layered structure described above
4. Add connections between components
5. Add labels to all components and connections
6. Use color coding as specified
7. Add the data flow paths with numbered arrows
8. Include a legend
9. Export as PNG with high resolution (at least 1200px wide)

## Example Layout Structure

```
+-------+     +-------+     +-------+     +-------+
| S3    |     | ofs   |     | Native|     | CLI   |
| Client|     | Client|     | Client|     | Client|
+-------+     +-------+     +-------+     +-------+
    |             |             |             |
    v             v             v             v
+-------------------------------------------------------+
|                     Protocol Layer                    |
+-------------------------------------------------------+
         |                                   |
         v                                   v
+------------------+              +------------------+
|  Ozone Manager   |<------------>| Storage Container|
|  (OM)            |              | Manager (SCM)    |
+------------------+              +------------------+
         |                                   |
         |                                   |
         v                                   v
+------------+   +------------+   +------------+
| Datanode 1 |   | Datanode 2 |   | Datanode n |
+------------+   +------------+   +------------+
                                        ^
                                        |
+-----------+                           |
|           |---------------------------+
|  Recon    |---------------------------+
|           |---------------------------+
+-----------+
```

Remember to show the data flow with numbered arrows as described above, and include the HA replication connections between OM instances and between SCM instances.