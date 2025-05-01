---
title: Understanding SCM Safe Mode in Apache Ozone
summary: A comprehensive guide to how Storage Container Manager (SCM) safe mode works, including initialization, exit conditions, configuration options, and administrator commands.
tags: [operations, administration, reliability]
---

# Understanding SCM Safe Mode

## Introduction

The Storage Container Manager (SCM) in Apache Ozone enters "Safe Mode" during startup to ensure system stability and data consistency. This protective state prevents certain operations until SCM verifies that the cluster has reached a stable state with sufficient resources available.

This document explains how SCM safe mode works, including its initialization process, exit conditions, configuration options, and administrator commands.

## Why Safe Mode Matters

Safe mode provides critical protection against data inconsistencies and potential data loss during system startup. Without safe mode, an SCM that doesn't have a complete view of the cluster could make incorrect decisions about container placement, replication, or pipeline creation.

Key benefits of SCM safe mode include:

-   **Data protection**: Ensures data containers are available before allowing operations
-   **Controlled startup**: Provides a consistent startup sequence for all components
-   **Recovery assurance**: Verifies sufficient replicas exist for data recovery
-   **System stability**: Confirms adequate resources before accepting client requests

## Safe Mode Lifecycle

### Initialization

When SCM starts, it automatically enters safe mode. The `SCMSafeModeManager` is initialized with the following components:

-   Container Manager
-   Pipeline Manager
-   Event Queue
-   Service Manager
-   SCM Context

```java
scmSafeModeManager = new SCMSafeModeManager(conf,
    containerManager, pipelineManager, eventQueue,
    serviceManager, scmContext);
```

The `SCMSafeModeManager` uses the `SafeModeRuleFactory` to load and initialize the safe mode exit rules. The `SafeModeRuleFactory` determines which rules to load based on the SCM configuration.

### Tracking State

SCM tracks the safe mode state using three atomic boolean flags:

| Flag                 | Description                                                    |
| -------------------- | -------------------------------------------------------------- |
| `inSafeMode`         | Indicates if SCM is currently in safe mode                   |
| `preCheckComplete`   | Indicates if initial pre-checks have completed                |
| `forceExitSafeMode`  | Indicates if safe mode was exited manually                    |

### Pre-Check Phase

Some operations can proceed after completing a set of "pre-check" rules, even if SCM remains in safe mode. The pre-check phase ensures that basic requirements are met before allowing certain services to start. Currently, the pre-check phase consists of the `DataNodeSafeModeRule`, which ensures that a minimum number of DataNodes have registered with SCM.

When all pre-check rules are validated:

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

-   The safe mode status is updated and propagated
-   SCM context is updated with the new status
-   Service manager is notified to start dependent services
-   SCM begins accepting all operations

## Safe Mode Exit Rules

SCM uses a rule-based system to determine when it's safe to exit safe mode. All rules must be satisfied before SCM automatically exits safe mode.

### 1. Container Rule

**Purpose**: Ensures sufficient containers have at least one replica reported

**Details**:

-   Tracks containers with at least one reported replica
-   Verifies that the percentage of containers with a replica reaches the configured threshold (default: 99%)
-   For EC containers, verifies each container has enough data nodes to meet minimum replication factor

**Events**: Processes `CONTAINER_REGISTRATION_REPORT` events

**Configuration**: `hdds.scm.safemode.threshold.pct` (default: 0.99)

### 2. DataNode Rule

**Purpose**: Ensures a minimum number of DataNodes have registered with SCM

**Details**:

-   Tracks registered DataNodes using their UUIDs
-   Verifies the number of registered DataNodes meets the minimum threshold

**Events**: Processes `NODE_REGISTRATION_CONT_REPORT` events

**Configuration**: `hdds.scm.safemode.min.datanode` (default: 1)

### 3. Healthy Pipeline Rule

**Purpose**: Ensures a sufficient percentage of pipelines are healthy

**Details**:

-   Tracks healthy Ratis/THREE pipelines
-   Verifies the percentage of healthy pipelines reaches the configured threshold

**Events**: Processes `OPEN_PIPELINE` events

**Configuration**: `hdds.scm.safemode.healthy.pipeline.pct` (default: 0.10)

### 4. One-Replica Pipeline Rule

**Purpose**: Ensures sufficient pipelines have at least one DataNode reported

**Details**:

-   Tracks pipelines with at least one DataNode that has reported
-   Verifies the percentage reaches the configured threshold

**Events**: Processes `PIPELINE_REPORT` events

**Configuration**: `hdds.scm.safemode.atleast.one.node.reported.pipeline.pct` (default: 0.90)

**Note:** The `HealthyPipelineRule` and `OneReplicaPipelineRule` are only enabled if the `hdds.scm.safemode.pipeline.availability.check` configuration is set to true.

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

| Parameter                                                    | Default | Description
