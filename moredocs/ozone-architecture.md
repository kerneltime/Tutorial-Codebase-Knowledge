# Apache Ozone Architecture Documentation

## Overview

Apache Ozone is a scalable, distributed object store designed for Hadoop and cloud-native environments. This document provides a comprehensive overview of Ozone's architecture, components, and their interactions.

## Architecture Diagram

![Apache Ozone Architecture](ozone-architecture.png)

## Core Components

### Ozone Manager (OM)

The Ozone Manager serves as the namespace manager for the Ozone system:

- **Purpose**: Manages the namespace hierarchy (volumes, buckets, and keys)
- **Functionality**:
  - Processes client requests for key operations (create, read, update, delete)
  - Allocates blocks for clients
  - Maintains metadata for objects
  - Manages ACLs and security policies
- **Storage**: Uses RocksDB for persistent storage of metadata
- **High Availability**: Uses Ratis (Apache Ratis is a Raft implementation) for HA

### Storage Container Manager (SCM)

SCM is the cluster manager and the central control service:

- **Purpose**: Manages containers, datanodes, and pipelines
- **Functionality**:
  - Allocates blocks and container space
  - Creates and manages pipelines for data replication
  - Tracks container states and replica distribution
  - Acts as certificate authority for security
  - Handles container replication and balancing
- **Storage**: Uses RocksDB for persistent storage of metadata
- **Heartbeats**: Receives regular heartbeats from datanodes with container reports

### Datanodes

Datanodes are the storage nodes in the Ozone cluster:

- **Purpose**: Store the actual data in containers
- **Functionality**:
  - Create and manage containers locally
  - Send regular heartbeats to SCM
  - Handle read/write requests from clients
  - Participate in replication protocols
  - Track space usage and report metrics
- **Storage**: Uses local filesystem to store container files
- **Container Management**: Keeps track of container states (open, closing, closed)

### Recon

Recon is the management and monitoring service for Ozone:

- **Purpose**: Provides holistic view of the cluster for administrators
- **Functionality**:
  - Web UI for monitoring cluster status
  - REST APIs for programmatic access
  - Collects metrics from all components
  - Maintains copies of OM and SCM databases
  - Integrates with Prometheus for alerting
- **Storage**: Uses local databases to store collected information

## Data Architecture

### Containers

Containers are the fundamental storage and replication units:

- **Size**: 5GB by default
- **States**: OPEN, CLOSING, CLOSED, etc.
- **Storage**: Each container is a self-contained unit with:
  - Container metadata (RocksDB)
  - Data chunks
  - Block metadata

### Data Organization

- **Volumes**: Top-level namespace unit (similar to S3 accounts)
- **Buckets**: Container for objects within volumes (similar to S3 buckets)
- **Keys**: Individual objects stored in buckets
- **Blocks**: Client-visible units of data
- **Containers**: Server-side storage units that hold blocks

## Protocol Support

Ozone supports multiple protocols for client access:

- **S3 Compatible**: S3 REST API for object operations
- **Ozone File System (ofs)**: Hadoop-compatible filesystem interface with unified namespace
- **Native API**: Java API for direct interaction
- **CSI Driver**: Container Storage Interface for Kubernetes

## Data Flow

### Write Path

1. Client requests to create a key from OM
2. OM allocates blocks by requesting from SCM
3. SCM returns pipeline information (set of datanodes)
4. Client writes data directly to datanodes in the pipeline
5. Datanodes use Ratis to replicate data for open containers
6. Client commits the key to OM after successful write

### Read Path

1. Client requests block locations for a key from OM
2. OM returns block IDs and locations
3. Client reads blocks directly from datanodes
4. For erasure-coded data, client may read from multiple datanodes and reconstruct

## Replication

Ozone provides two replication strategies:

- **Ratis Replication**: For open containers, using Raft protocol
- **Container Replication**: For closed containers, using async copy
- **Erasure Coding**: For data protection with storage efficiency (optional)

## Security

Ozone security is based on:

- **Kerberos**: Authentication
- **TLS**: Transport security
- **Ranger**: Authorization and access control
- **Transparent Data Encryption**: Data at rest encryption

## High Availability

- **OM HA**: Multiple OM instances with Ratis consensus
- **SCM HA**: Multiple SCM instances with leader election
- **Datanode Redundancy**: Multiple replicas of data

## Advanced Features

- **S3 Multi-tenancy**: Isolation of accounts
- **Snapshots**: Point-in-time snapshots for data protection
- **Quotas**: Volume and bucket quota management
- **Trash**: Soft-delete and recovery of objects
- **Topology-aware placement**: Network topology awareness for data placement

## Monitoring and Management

- **Metrics**: Prometheus integration
- **REST API**: HTTP endpoints for management
- **Recon UI**: Web console for administrators
- **CLI**: Command-line interface for operations

## Conclusion

Ozone's architecture provides a scalable, resilient object store suitable for both Hadoop and cloud-native workloads. The clean separation of namespace (OM) from block management (SCM) allows it to scale to billions of objects while maintaining high performance.

The use of containers as the fundamental storage unit and replication via Ratis provides both performance and data safety. Multiple client protocols enable wide integration with existing applications and systems.