# Apache Ozone Architecture

Apache Ozone is a distributed object store designed for scalability, redundancy, and cloud-native environments. Ozone separates namespace management and block space management, which helps it scale to billions of objects.

## Core Architecture Components

![Architecture Diagram](https://ozone.apache.org/docs/concept/ozoneBlockDiagram.png)

The Ozone architecture consists of the following key components:

### Ozone Manager (OM)

The Ozone Manager is the namespace manager responsible for:

* Managing the object namespace (volumes, buckets, and keys)
* Storing metadata about objects and directories
* Processing client requests for namespace operations
* Authenticating clients and authorizing access
* Managing volume and bucket operations

When clients write data, OM allocates blocks and provides block tokens that authorize clients to write to specific blocks.

### Storage Container Manager (SCM)

Storage Container Manager is the core metadata service that provides a distributed block layer for Ozone. SCM is responsible for:

* Creating and managing containers (the main replication units)
* Acting as the cluster manager
* Serving as a Certificate Authority for the cluster
* Allocating blocks and assigning them to DataNodes
* Tracking all block replicas
* Managing DataNode membership
* Ensuring data replication for high availability

SCM maintains the mapping of containers to DataNodes and organizes DataNodes into pipeline groups for replication.

### DataNodes (DN)

DataNodes are the worker nodes that store the actual data in containers. They:

* Store data blocks on local disks
* Serve data to clients
* Replicate data for redundancy
* Report storage usage and container status to SCM
* Execute commands received from SCM

### Recon

Recon is the management and monitoring server for Ozone that:

* Collects information from all components (OM, SCM, DataNodes)
* Provides a unified management API and UI
* Enables administrators to monitor cluster health and usage
* Offers troubleshooting tools and insights

### Protocol Bus

The Protocol Bus allows Ozone to be extended via other protocols:

* Currently supports S3 protocol through S3 Gateway
* Enables implementation of new file system or object store protocols
* Translates external protocols to Ozone's native protocol

## Data Organization

Ozone organizes data in a hierarchical structure:

1. **Volumes**: Similar to home directories, created by administrators
   * Used for storage accounting
   * Contain multiple buckets

2. **Buckets**: Containers for objects, created by users
   * Similar to directories but with object store semantics
   * Support policies like versioning and replication

3. **Keys**: The actual data objects stored in buckets
   * Composed of multiple blocks
   * Blocks are stored in containers on DataNodes

## Data Flow

### Write Path

1. Client requests to create a key from Ozone Manager
2. OM allocates blocks and provides block tokens to the client
3. Client uses these tokens to write data directly to DataNodes
4. SCM organizes DataNodes into pipelines for data replication
5. Once data is written, client commits the key to OM
6. OM updates its metadata to reflect the new key

### Read Path

1. Client requests location information for a key from OM
2. OM provides block locations and security tokens
3. Client reads data directly from DataNodes
4. If a DataNode is unavailable, client retries with other replicas

## Replication and Consistency

Ozone uses the Apache Ratis implementation of the Raft consensus algorithm for:

* Replicating metadata in OM and SCM for high availability
* Ensuring consistency when data is modified at the DataNodes
* Providing strong consistency guarantees for object operations

## Functional Layers

Ozone can be viewed in terms of functional layers:

1. **Metadata Management Layer**: Composed of Ozone Manager and Storage Container Manager
2. **Data Storage Layer**: Made up of DataNodes managed by SCM
3. **Replication Layer**: Provided by Ratis for metadata and data consistency
4. **Management Layer**: Recon provides unified management capabilities
5. **Protocol Layer**: Enables access via different protocols (Native, S3, etc.)

## Security

Ozone has a comprehensive security model:

* Certificate-based authentication for internal components
* Kerberos integration for client authentication
* ACL-based authorization
* Ranger integration for advanced access control
* Block tokens for secure data access
* TDE and on-wire encryption for data protection

This architecture makes Ozone highly scalable, reliable, and suitable for cloud-native environments while providing the security and consistency guarantees required for enterprise workloads.