## SCM Safe Mode Handling in Apache Ozone

SCM (Storage Container Manager) safe mode is a crucial mechanism in Apache Ozone that ensures data safety during startup. It allows the system to reach a stable state before becoming fully functional, preventing data loss or corruption. This document provides a detailed explanation of SCM safe mode, including when SCM enters and exits safe mode, the rules governing its behavior, and relevant configuration parameters.

### When Does SCM Enter Safe Mode?

SCM enters safe mode automatically during startup if the `hdds.scm.safemode.enabled` configuration property is set to `true`. This property is enabled by default.

### When Does SCM Exit Safe Mode?

SCM exits safe mode when all the configured safe mode exit rules are satisfied. These rules define the conditions that must be met before SCM can be considered stable and ready to handle client requests. The following rules are currently implemented:

1.  **ContainerSafeModeRule:** This rule checks if a sufficient percentage of containers have been reported to SCM with at least one replica. The required percentage is controlled by the `hdds.scm.safemode.threshold.pct` configuration property. This rule differentiates between RATIS and EC containers. For RATIS containers, it checks if at least one replica has been reported. For EC containers, it checks if at least N replicas have been reported, where N is the number of data blocks in the EC scheme.
2.  **HealthyPipelineSafeModeRule:** This rule checks if a sufficient percentage of pipelines are healthy. A healthy pipeline is a pipeline that is fully functional and can be used for data storage. The required percentage is controlled by the `hdds.scm.safemode.healthy.pipeline.threshold.pct` configuration property. This rule only considers RATIS pipelines with a replication factor of THREE. The minimum number of datanodes required for SCM to exit safe mode, controlled by the `hdds.scm.safemode.min.datanode` property, indirectly affects the minimum number of healthy pipelines.
3.  **OneReplicaPipelineSafeModeRule:** This rule checks if a sufficient percentage of pipelines have at least one datanode reporting. This ensures that there is at least one replica available for read for all open containers when SCM exits safe mode. The required percentage is controlled by the `hdds.scm.safemode.one.node.reported.pipeline.pct` configuration property. This rule only considers RATIS pipelines with a replication factor of THREE.

### Safe Mode Exit Process

The following steps are involved in the SCM safe mode exit process:

1.  **Initialization:** When SCM starts, the `SCMSafeModeManager` is initialized. This class is responsible for managing the overall safe mode state.
2.  **Rule Loading:** The `SCMSafeModeManager` loads the configured safe mode exit rules.
3.  **Event Handling:** Each `SafeModeExitRule` registers itself as a listener for a specific event. For example, the `ContainerSafeModeRule` listens for the `CONTAINER_REGISTRATION_REPORT` event, which is triggered when a datanode reports its containers to SCM.
4.  **Validation:** When an event occurs, the corresponding `SafeModeExitRule` is notified. The rule then calls its `validate` method to determine if its condition has been met.
5.  **Exit:** When all the `SafeModeExitRule`s have been satisfied, the `SCMSafeModeManager` exits safe mode.

### Configuration Parameters

The following configuration parameters control the behavior of SCM safe mode:

*   `hdds.scm.safemode.enabled`: Enables or disables SCM safe mode. Default: `true`.
*   `hdds.scm.safemode.threshold.pct`: The required percentage of containers with at least one replica for the `ContainerSafeModeRule`. Default: `0.0`.
*   `hdds.scm.safemode.healthy.pipeline.threshold.pct`: The required percentage of healthy pipelines for the `HealthyPipelineSafeModeRule`. Default: `0.0`.
*   `hdds.scm.safemode.min.datanode`: The minimum number of datanodes required for SCM to exit safe mode. This indirectly affects the minimum number of healthy pipelines. Default: `0`.
*   `hdds.scm.safemode.one.node.reported.pipeline.pct`: The required percentage of pipelines with at least one datanode reporting for the `OneReplicaPipelineSafeModeRule`. Default: `0.0`.

### Troubleshooting

If SCM fails to exit safe mode, check the following:

1.  **Configuration:** Ensure that the safe mode configuration parameters are set correctly.
2.  **Datanode Registration:** Ensure that all datanodes have registered with SCM.
3.  **Container Reporting:** Ensure that all datanodes have reported their containers to SCM.
4.  **Pipeline Health:** Ensure that a sufficient number of pipelines are healthy.

You can use the SCM CLI to monitor the status of safe mode and the individual exit rules. The CLI provides information on the current state of each rule and the number of resources that have been reported.

### Importance for Data Safety

SCM safe mode is essential for data safety in Apache Ozone. By ensuring that the system reaches a stable state before becoming fully functional, it prevents data loss or corruption during startup. It is crucial to configure safe mode correctly and to monitor its status to ensure that SCM exits safe mode in a timely manner.