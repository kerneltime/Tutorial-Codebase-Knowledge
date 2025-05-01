# SCM Safe Mode Handling in Apache Ozone

## Overview

Safe Mode is a protective state that the Storage Container Manager (SCM) enters during startup to ensure system stability before becoming fully operational. During safe mode, SCM restricts certain operations (like creating new blocks or containers) until it has verified that the cluster is in a consistent and healthy state. This prevents data loss or corruption during startup when the system might not have a complete view of the cluster state.

## Safe Mode Initialization

1.  **Startup Process**: When SCM starts, it initializes `SCMSafeModeManager` in the `StorageContainerManager` constructor:

    ```java
    scmSafeModeManager = new SCMSafeModeManager(conf,
        containerManager, pipelineManager, eventQueue,
        serviceManager, scmContext);
    ```

2.  **Configuration**: Safe mode is controlled by several configuration parameters:

    *   `hdds.scm.safemode.enabled` (default: true) - Enables/disables safe mode. If disabled, SCM will skip the safe mode process and immediately become fully operational.
    *   `hdds.scm.safemode.min.datanode` (default: 1) - Minimum number of DataNodes required to register with SCM before exiting safe mode. This ensures that SCM has a basic quorum of DataNodes available.
    *   `hdds.scm.safemode.threshold.pct` (default: 0.99) - Percentage of containers that must have at least one replica reported before exiting safe mode. This ensures that a sufficient amount of data is available.
    *   `hdds.scm.safemode.healthy.pipeline.pct` (default: 0.10) - Percentage of healthy pipelines required before exiting safe mode. This ensures that the write path is ready before allowing write operations.
    *   `hdds.scm.safemode.atleast.one.node.reported.pipeline.pct` (default: 0.90) - Percentage of pipelines that must have at least one DataNode reported before exiting safe mode. This ensures that there's at least one copy of all data available, which is critical for recovery operations.
3.  **State Tracking**: Safe mode state is tracked using atomic boolean flags:

    *   `inSafeMode` - Indicates if SCM is in safe mode (initialized to true).
    *   `preCheckComplete` - Indicates if initial pre-checks have completed.
    *   `forceExitSafeMode` - Indicates if safe mode was exited manually.

## Safe Mode Exit Rules

SCM uses a set of rules to determine when it's safe to exit safe mode. All rules must be satisfied before SCM can automatically exit safe mode.

1.  **ContainerSafeModeRule**: Ensures that a sufficient percentage of containers have at least one replica reported. This is critical for data availability.

    *   Tracks containers with at least one reported replica.
    *   Verifies that the percentage of containers with a replica reaches the threshold (default: 99%).
    *   For EC containers, verifies that each container has enough data nodes to meet the minimum replication factor.
    *   Located in: `ContainerSafeModeRule.java`

    **Ratis vs. EC Containers:** Apache Ozone supports two types of containers: Ratis and Erasure Coded (EC). Ratis containers use a traditional replication scheme, where each replica is a full copy of the data. EC containers use an erasure coding scheme, where data is split into fragments and parity fragments are generated. EC containers provide better storage efficiency but require more DataNodes to be available for data recovery.

    **Minimum Replica for EC Containers:** The minimum replica for EC containers is the number of data nodes in the replication config. This ensures that enough data fragments are available to reconstruct the original data in case of DataNode failures.

    **ecContainerDNsMap:** This data structure is used to track the DataNodes that have reported replicas for EC containers. This is necessary because EC containers require multiple DataNodes to be available for data recovery.

2.  **DataNodeSafeModeRule**: Ensures a minimum number of DataNodes have registered with SCM.

    *   Tracks registered DataNodes using their UUIDs.
    *   Verifies that the number of registered DataNodes meets the minimum threshold (default: 1).
    *   Located in: `DataNodeSafeModeRule.java`

    **registeredDnSet:** This data structure is used to track the DataNodes that have registered with SCM. A set is used to ensure that each DataNode is only counted once, even if it registers multiple times.

    **HDDS\_SCM\_SAFEMODE\_MIN\_DATANODE:** This configuration parameter determines the minimum number of DataNodes that must register with SCM before safe mode can be exited. This ensures that SCM has a basic quorum of DataNodes available to manage the cluster.

3.  **HealthyPipelineSafeModeRule**: Ensures a sufficient percentage of pipelines are healthy.

    *   Tracks healthy Ratis/THREE pipelines.
    *   Verifies that the percentage of healthy pipelines reaches the threshold (default: 10%).
    *   Located in: `HealthyPipelineSafeModeRule.java`

    **Significance of Healthy Pipelines:** Healthy pipelines are necessary for SCM to be able to write data to the cluster. A healthy pipeline is one that has a sufficient number of DataNodes available and is not experiencing any errors.

    **HDDS\_SCM\_SAFEMODE\_HEALTHY\_PIPELINE\_THRESHOLD\_PCT:** This configuration parameter determines the required percentage of healthy pipelines before safe mode can be exited.

    **Ratis/THREE Pipelines:** This rule only considers Ratis/THREE pipelines because these are the most common type of pipeline used in Apache Ozone. Other types of pipelines may be added in the future.

    **Criteria for a Healthy Pipeline:** A pipeline is considered healthy if it meets the following criteria:

    *   All DataNodes in the pipeline are alive and responsive.
    *   The pipeline is not experiencing any errors.
    *   The pipeline has a sufficient number of DataNodes available to meet the replication requirements.

    **FinalizationManager.shouldCreateNewPipelines:** This method checks whether the SCM is in the process of upgrading. If so, new pipelines should not be created, and the healthy pipeline safe mode rule is bypassed.

4.  **OneReplicaPipelineSafeModeRule**: Ensures that for a sufficient percentage of pipelines, at least one DataNode has reported.

    *   Tracks pipelines with at least one DataNode that has reported.
    *   Verifies that the percentage reaches the threshold (default: 90%).
    *   Located in: `OneReplicaPipelineSafeModeRule.java`

    **Purpose of Ensuring at Least One DataNode Has Reported:** This ensures that there's at least one copy of all data available, which is critical for recovery operations.

    **HDDS\_SCM\_SAFEMODE\_ONE\_NODE\_REPORTED\_PIPELINE\_PCT:** This configuration parameter determines the required percentage of pipelines with at least one DataNode reported before safe mode can be exited.

    **Ratis/THREE Pipelines:** This rule only considers Ratis/THREE pipelines because these are the most common type of pipeline used in Apache Ozone. Other types of pipelines may be added in the future.

    **Criteria for a DataNode to be Considered "Reported":** A DataNode is considered "reported" for a pipeline if it has sent a pipeline report to SCM indicating that it is part of the pipeline.

    **oldPipelineIDSet and reportedPipelineIDSet:** These data structures are used to track the pipelines that have been processed. The `oldPipelineIDSet` contains all pipelines that were open when the rule was initialized, and the `reportedPipelineIDSet` contains the pipelines for which at least one DataNode has reported.

## Rule Validation Process

1.  **Event Handling**: Each rule registers as a handler for specific events:

    *   `ContainerSafeModeRule`: Handles `CONTAINER_REGISTRATION_REPORT` events
    *   `DataNodeSafeModeRule`: Handles `NODE_REGISTRATION_CONT_REPORT` events
    *   `HealthyPipelineSafeModeRule`: Handles `OPEN_PIPELINE` events
    *   `OneReplicaPipelineSafeModeRule`: Handles `PIPELINE_REPORT` events

2.  **Rule Processing**: When events are received, each rule:

    *   Processes the event (updates its internal state)
    *   Checks if its validation condition is met
    *   If the condition is met, notifies the `SCMSafeModeManager`

3.  **Rule Lifecycle**:

    *   When a rule is initialized, it calculates the threshold it needs to meet
    *   The rule processes events as they arrive, updating its state
    *   When the rule's condition is met, it calls `validateSafeModeExitRules` on the safe mode manager
    *   After validation, the rule cleans up its resources

## Pre-Check Phase

1.  **Purpose**: Some rules are designated as "pre-check" rules. These rules must be satisfied before certain operations can proceed, even if SCM is still in safe mode.
2.  **Completion**: When all pre-check rules are validated, the `preCheckComplete` flag is set to true, and the safe mode status is emitted.
3.  **Service Notification**: When pre-checks complete, the `SCMServiceManager` is notified with the `PRE_CHECK_COMPLETED` event, which allows certain services to start.

## Safe Mode Exit Process

1.  **Automatic Exit**: SCM exits safe mode when all exit rules are satisfied:

    ```java
    if (validatedRules.size() == exitRules.size()) {
      // All rules are satisfied, we can exit safe mode.
      LOG.info("ScmSafeModeManager, all rules are successfully validated");
      exitSafeMode(eventQueue, false);
    }
    ```

2.  **Manual Exit**: SCM can be forced out of safe mode using the `exitSafeMode` method:

    ```java
    public void exitSafeMode(EventPublisher eventQueue, boolean force) {
      LOG.info("SCM exiting safe mode.");
      setPreCheckComplete(true);
      setInSafeMode(false);
      setForceExitSafeMode(force);
      emitSafeModeStatus();
    }
    ```

3.  **Status Notification**: When SCM exits safe mode:

    *   The `inSafeMode` flag is set to false
    *   The safe mode status is emitted through the event queue
    *   The `SCMContext` is updated with the new status
    *   The `SCMServiceManager` is notified to start services that were waiting for safe mode to exit

## Rule Refresh Mechanism

1.  **Periodic Refresh**: Rules are refreshed to reflect the current system state:

    ```java
    public void refresh() {
      if (inSafeMode.get()) {
        exitRules.values().forEach(rule -> {
          rule.refresh(true);
        });
      }
    }
    ```

2.  **Validate After Refresh**: After refreshing, rules are re-validated:

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

## Safe Mode Status and Monitoring

1.  **Status Reporting**: SCM provides methods to check the current safe mode status:

    *   `getInSafeMode()` - Returns whether SCM is in safe mode
    *   `getPreCheckComplete()` - Returns whether pre-checks are complete
    *   `getRuleStatus()` - Returns the status of all safe mode exit rules

2.  **Status Emission**: SCM emits safe mode status events which contain:

    *   Current safe mode state
    *   Whether pre-checks have completed
    *   Whether safe mode was forcibly exited

    **SafeModeStatus Class:** This class encapsulates the current safe mode state, whether pre-checks have completed, and whether safe mode was forcibly exited. This information is used by other components of SCM to determine how to behave.

3.  **Metrics**: SCM tracks various metrics related to safe mode:

    *   Number of containers with at least one replica reported
    *   Number of healthy pipelines
    *   Thresholds for each rule
    *   Current counts for each rule

## Practical Implications for Data Safety

1.  **Data Protection**: Safe mode ensures that enough containers, pipelines, and DataNodes are available before allowing operations that could modify data. This protects against data loss during startup when the system might not have a complete view of the cluster state.
2.  **Progressive Service Startup**: The pre-check mechanism allows some services to start once basic checks have passed, even before safe mode fully exits. This enables a progressive startup of the system.
3.  **Recovery Assurance**: By requiring that containers have at least one replica before exiting safe mode, SCM ensures that there's at least one copy of all data available, which is critical for recovery operations.
4.  **Pipeline Health**: By verifying pipeline health, SCM ensures that the write path is ready before allowing write operations, which prevents data inconsistencies.

## Configuration Best Practices

1.  **Minimum DataNodes**: Set `hdds.scm.safemode.min.datanode` based on your cluster size and reliability requirements. For production clusters, use a value that ensures at least basic quorum.
2.  **Container Threshold**: The default of 99% (`hdds.scm.safemode.threshold.pct`) ensures nearly all containers have at least one replica. Lower this value only if you're willing to accept the risk of some containers being unavailable.
3.  **Pipeline Thresholds**: The defaults (10% for healthy pipelines, 90% for pipelines with at least one node) balance safety with startup speed. Adjust based on your reliability needs.
4.  **Safe Mode Wait Time**: After SCM exits safe mode, it waits a configurable amount of time (`hdds.scm.wait.time.after.safemode.exit`, default: 5 minutes) before certain operations. This gives components time to fully initialize.

## Troubleshooting

If SCM is taking a long time to exit safe mode, you can use the SCM CLI to check the status of the safe mode exit rules. The `ozone scm safemode get` command will display the status of each rule, including the current value and the threshold value. This can help you identify which rule is preventing SCM from exiting safe mode.

You can also check the SCM logs for any errors or warnings related to safe mode. The logs may contain information about why a particular rule is not being satisfied.

If you need to force SCM out of safe mode, you can use the `ozone scm safemode exit` command. However, this should only be done as a last resort, as it can potentially lead to data loss or corruption.

## Conclusion

SCM safe mode is a critical mechanism for ensuring data safety and system stability during Ozone startup. It works by verifying that the system has reached a stable state with sufficient resources (DataNodes, containers, and pipelines) before allowing normal operations.

The exit from safe mode happens automatically when all configured thresholds are met, or can be forced manually if necessary. The use of multiple rules with different criteria ensures a comprehensive check of system health before exit.

Understanding these mechanisms is crucial for operators managing Ozone clusters, as it directly impacts cluster availability, data safety, and recovery processes.