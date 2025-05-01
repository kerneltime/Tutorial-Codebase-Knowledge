---
title: Administering SCM Safe Mode in Apache Ozone
summary: A comprehensive guide for Apache Ozone administrators on managing Storage Container Manager (SCM) safe mode, including initialization, exit conditions, configuration options, and troubleshooting.
tags: [operations, administration, reliability, safemode, scm]
---

# Administering SCM Safe Mode

## Introduction

The Storage Container Manager (SCM) in Apache Ozone enters "Safe Mode" during startup to ensure system stability and data consistency. This protective state prevents certain operations until SCM verifies that the cluster has reached a stable state with sufficient resources available.

This document explains how SCM safe mode works from an administrator's perspective, including its initialization process, exit conditions, configuration options, monitoring, and troubleshooting.

## Why Safe Mode Matters

Safe mode provides critical protection against data inconsistencies and potential data loss during system startup. Without safe mode, an SCM that doesn't have a complete view of the cluster could make incorrect decisions about container placement, replication, or pipeline creation.

Key benefits of SCM safe mode include:

*   **Data protection**: Ensures data containers are available before allowing operations
*   **Controlled startup**: Provides a consistent startup sequence for all components
*   **Recovery assurance**: Verifies sufficient replicas exist for data recovery
*   **System stability**: Confirms adequate resources before accepting client requests

## Safe Mode Lifecycle

### Initialization

When SCM starts, it automatically enters safe mode, initializing the `SCMSafeModeManager` with the following components:

*   Container Manager: Manages the lifecycle of containers in the cluster.
*   Pipeline Manager: Manages the creation and maintenance of pipelines, which are used for data replication.
*   Event Queue: A mechanism for asynchronous communication between SCM components.
*   Service Manager: Manages the lifecycle of SCM services.
*   SCM Context: Provides access to SCM configuration and state.

```java
scmSafeModeManager = new SCMSafeModeManager(conf,
    containerManager, pipelineManager, eventQueue,
    serviceManager, scmContext);
```

### Tracking State

SCM tracks the safe mode state using three atomic boolean flags:

| Flag              | Description                                                                 |
| ----------------- | --------------------------------------------------------------------------- |
| `inSafeMode`      | Indicates if SCM is currently in safe mode                                |
| `preCheckComplete` | Indicates if initial pre-checks have completed                             |
| `forceExitSafeMode` | Indicates if safe mode was exited manually                               |

### Pre-Check Phase

Some operations can proceed after completing a set of "pre-check" rules, even if SCM remains in safe mode. These pre-check rules are a subset of the exit rules and are designed to allow certain services to start before SCM has fully exited safe mode. When all pre-check rules are validated:

1.  The `preCheckComplete` flag is set to true
2.  Safe mode status is emitted to notify components
3.  The `SCMServiceManager` receives a `PRE_CHECK_COMPLETED` event
4.  Certain services can start while SCM remains in safe mode

### Exit Process

SCM exits safe mode in one of two ways:

**Automatic Exit**:
When all exit rules are satisfied, SCM automatically exits safe mode:

```java
if (validatedRules.size() == exitRules.size()) {
  LOG.info("ScmSafeModeManager, all rules are successfully validated");
  exitSafeMode(eventQueue, false);
}
```

**Manual Exit**:
An administrator can force SCM to exit safe mode using admin commands, which calls:

```java
public void exitSafeMode(EventPublisher eventQueue, boolean force) {
  LOG.info("SCM exiting safe mode.");
  setPreCheckComplete(true);
  setInSafeMode(false);
  setForceExitSafeMode(force);
  emitSafeModeStatus();
}
```

Upon exiting safe mode:

*   The safe mode status is updated and propagated
*   SCM context is updated with the new status
*   Service manager is notified to start dependent services
*   SCM begins accepting all operations

## Safe Mode Exit Rules

SCM uses a rule-based system to determine when it's safe to exit safe mode. All rules must be satisfied before SCM automatically exits safe mode. These rules ensure that the cluster has reached a stable state with sufficient resources available before SCM allows normal operations.

### 1. Container Rule

**Purpose**: Ensures sufficient containers have at least one replica reported. This is critical for data availability.

**Details**:

*   Tracks containers with at least one reported replica
*   Verifies that the percentage of containers with a replica reaches the configured threshold (default: 99%)
*   For EC containers, verifies each container has enough data nodes to meet the minimum replication factor

**Events**: Processes `CONTAINER_REGISTRATION_REPORT` events. This event is triggered when a DataNode registers with SCM and reports the containers it is storing.

**Configuration**: `hdds.scm.safemode.threshold.pct` (default: 0.99). This parameter determines the percentage of containers that must have at least one replica reported before SCM can exit safe mode.

**Ratis vs. EC Containers:** Apache Ozone supports two types of containers: Ratis and Erasure Coded (EC). Ratis containers use a traditional replication scheme, where each replica is a full copy of the data. EC containers use an erasure coding scheme, where data is split into fragments and parity fragments are generated. EC containers provide better storage efficiency but require more DataNodes to be available for data recovery. The Container Rule ensures that both Ratis and EC containers have sufficient replicas before SCM exits safe mode.

### 2. DataNode Rule

**Purpose**: Ensures a minimum number of DataNodes have registered with SCM. This ensures that SCM has a basic quorum of DataNodes available to manage the cluster.

**Details**:

*   Tracks registered DataNodes using their UUIDs
*   Verifies the number of registered DataNodes meets the minimum threshold

**Events**: Processes `NODE_REGISTRATION_CONT_REPORT` events. This event is triggered when a DataNode registers with SCM.

**Configuration**: `hdds.scm.safemode.min.datanode` (default: 1). This parameter determines the minimum number of DataNodes that must register with SCM before SCM can exit safe mode.

### 3. Healthy Pipeline Rule

**Purpose**: Ensures a sufficient percentage of pipelines are healthy. This ensures that the write path is ready before allowing write operations.

**Details**:

*   Tracks healthy Ratis/THREE pipelines
*   Verifies the percentage of healthy pipelines reaches the configured threshold

**Events**: Processes `OPEN_PIPELINE` events. This event is triggered when a new pipeline is created.

**Configuration**: `hdds.scm.safemode.healthy.pipeline.pct` (default: 0.10). This parameter determines the percentage of pipelines that must be healthy before SCM can exit safe mode.

**Ratis/THREE Pipelines:** This rule only considers Ratis/THREE pipelines because these are the most common type of pipeline used in Apache Ozone. Other types of pipelines may be added in the future. A Ratis/THREE pipeline is a pipeline that uses the Ratis consensus protocol and has a replication factor of 3.

### 4. One-Replica Pipeline Rule

**Purpose**: Ensures sufficient pipelines have at least one DataNode reported. This ensures that there's at least one copy of all data available, which is critical for recovery operations.

**Details**:

*   Tracks pipelines with at least one DataNode that has reported
*   Verifies the percentage reaches the configured threshold

**Events**: Processes `PIPELINE_REPORT` events. This event is triggered when a DataNode sends a report to SCM about the pipelines it is participating in.

**Configuration**: `hdds.scm.safemode.atleast.one.node.reported.pipeline.pct` (default: 0.90). This parameter determines the percentage of pipelines that must have at least one DataNode reported before SCM can exit safe mode.

## Rule Processing

When events are received, each rule:

1.  Processes the event to update its internal state
2.  Checks if its validation condition is met
3.  If the condition is met, notifies the `SCMSafeModeManager`
4.  When all rules are validated, SCM exits safe mode

Rules are periodically refreshed to reflect the current system state:

```java
public void refreshAndValidate() {
  if (inSafeMode.get()) {
    exitRules.values().forEach(rule -> {
      rule.refresh(false);
      if (rule.validate() && inSafeMode.get()) {
        validateSafeModeExitRules(rule.getRuleName(), eventPublisher);
        rule.cleanup();
      }
    });
  }
}
```

## Configuration Options

SCM safe mode behavior can be customized through these configuration parameters:

