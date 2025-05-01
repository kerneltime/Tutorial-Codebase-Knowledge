# Apache Ozone Web UI Guide

This guide provides comprehensive documentation on the web user interfaces available for each Apache Ozone service, including their functionality, access methods, and configuration options.

## Overview

Each Ozone service provides a web UI that allows administrators to monitor, troubleshoot, and manage the service. The web UIs offer a range of features, from viewing metrics and logs to changing logging levels and performing administrative operations.

## Common Features Across All Web UIs

All Ozone service web UIs include the following common features:

| Feature | URL Path | Description |
|---------|----------|-------------|
| JMX Metrics | `/jmx` | JSON-formatted JMX metrics for monitoring |
| Configuration | `/conf` | View current configuration parameters |
| Log Level | `/logLevel` | View and modify logging levels |
| Prometheus Metrics | `/prom` | Metrics in Prometheus format for external monitoring |
| Stack Traces | `/stacks` | Thread stack traces for debugging |
| Profiling | `/prof` | CPU/Memory profiling via async-profiler (when enabled) |

### How to Access Web UIs

Each service exposes its web UI on a specific HTTP port:

| Service | Default Port | Configuration Parameter |
|---------|-------------|-------------------------|
| Ozone Manager (OM) | 9874 (HTTP), 9875 (HTTPS) | `ozone.om.http-address`, `ozone.om.https-address` |
| Storage Container Manager (SCM) | 9876 (HTTP) | `ozone.scm.http-address` |
| DataNode | 9882 (HTTP) | | 
| S3 Gateway | 9878 (HTTP), 19878 (HTTPS) | |
| Recon | 9888 (HTTP), 9889 (HTTPS) | `ozone.recon.http-address`, `ozone.recon.https-address` |
| HTTP FileSystem Gateway | 14000 (HTTP) | |

## Ozone Manager (OM) Web UI

The OM web UI provides insights into the Ozone object store metadata operations.

### Main Features

- **Metrics Dashboard**: View key performance indicators for volume, bucket, and key operations
- **RPC Service Information**: Monitor RPC performance metrics, including request latencies and queue sizes
- **Database Checkpoint**: Create and download OM database checkpoints using `/dbCheckpoint`
- **Service List**: Access service list information using `/serviceList`

### Example Usage

1. **View OM metrics**: Navigate to `http://om-host:9874/`
2. **Check configuration**: Visit `http://om-host:9874/conf`
3. **Modify log levels**: Go to `http://om-host:9874/logLevel`
4. **Download a checkpoint**: Access `http://om-host:9874/dbCheckpoint`

## Storage Container Manager (SCM) Web UI

The SCM web UI provides visibility into container management and datanode health.

### Main Features

- **Container Metrics**: Monitor container counts by state (open, closing, closed, etc.)
- **Pipeline Information**: View Ratis pipeline statistics and health
- **Datanode Management**: See datanode status and health metrics
- **Database Checkpoint**: Create and download SCM database checkpoints using `/scm/dbCheckpoint`

### Example Usage

1. **View SCM dashboard**: Navigate to `http://scm-host:9876/`
2. **Check datanode status**: Access nodes information in the dashboard
3. **Monitor container metrics**: View container status on the main page
4. **Download a checkpoint**: Go to `http://scm-host:9876/scm/dbCheckpoint`

## DataNode Web UI

The DataNode web UI provides information about data storage and container operations on a specific node.

### Main Features

- **Volume Usage**: Track disk space utilization across volumes
- **Container Statistics**: Monitor container operations and status
- **Pipeline Information**: View pipelines this datanode participates in
- **RPC Metrics**: Analyze RPC performance statistics

### Example Usage

1. **View DataNode dashboard**: Navigate to `http://datanode-host:9882/`
2. **Check volume usage**: Access volume information on the dashboard
3. **Monitor container operations**: View container metrics on the main page

## S3 Gateway Web UI

The S3 Gateway web UI provides information about the S3-compatible interface.

### Main Features

- **Administrative Interface**: Manage S3 compatibility settings
- **S3 Bucket Operations**: Monitor S3 bucket operations statistics
- **Authentication Statistics**: Track authentication and authorization metrics

### Example Usage

1. **View S3 Gateway dashboard**: Navigate to `http://s3g-host:9878/`
2. **Get S3 credentials**: For Kerberos-authenticated users, use `ozone s3 getsecret` to obtain AWS-compatible credentials

## Recon Web UI (Management Console)

Recon provides a comprehensive management console for Ozone clusters with the most advanced web UI.

### Main Features

- **Cluster Overview Dashboard**: Comprehensive view of cluster health
- **Container Management**: Advanced container tracking, including missing and unhealthy containers
- **Datanode Management**: Detailed view of datanode health and statistics
- **Pipeline Monitoring**: Track pipeline states and health
- **Volume and Bucket Metrics**: Monitor storage usage across volumes and buckets
- **Namespace Analysis**: Detailed namespace usage statistics and visualization
- **File Size Distribution**: Analyze file counts by size ranges
- **Prometheus Metrics Visualization**: Built-in visualization of key metrics

### API Endpoints

Recon exposes a rich REST API for programmatic access to cluster information:

#### Cluster State Endpoints
- `/api/v1/clusterState`: Overall cluster state
- `/api/v1/datanodes`: List of all datanodes
- `/api/v1/pipelines`: List of all pipelines

#### Container Endpoints (Admin Only)
- `/api/v1/containers`: All containers information
- `/api/v1/containers/missing`: Missing containers
- `/api/v1/containers/unhealthy`: Unhealthy containers
- `/api/v1/containers/{id}/keys`: Keys in a specific container

#### Namespace Endpoints (Admin Only)
- `/api/v1/namespace/summary`: Summary of entities under a path
- `/api/v1/namespace/du`: Disk usage information
- `/api/v1/namespace/quota`: Quota information
- `/api/v1/namespace/dist`: File size distribution

#### Volume and Bucket Endpoints (Admin Only)
- `/api/v1/volumes`: List all volumes
- `/api/v1/buckets`: List all buckets

#### Utilization Endpoints
- `/api/v1/utilization/fileCount`: File counts by size
- `/api/v1/utilization/containerCount`: Container counts by size

#### Metrics Endpoint
- `/api/v1/metrics/query`: Proxy for Prometheus metrics

### Swagger Documentation
Recon provides Swagger documentation for its API at `/swagger-ui`.

### Example Usage

1. **View Recon dashboard**: Navigate to `http://recon-host:9888/`
2. **Check container status**: Go to the Containers tab
3. **Monitor datanode health**: Visit the Datanodes tab
4. **View API documentation**: Access `http://recon-host:9888/swagger-ui`

## HTTP FileSystem Gateway Web UI

The HTTP FileSystem Gateway web UI provides information about the HTTP-based access to Ozone file systems.

### Main Features

- **Filesystem Operations**: Monitor file system operations statistics
- **Endpoint Health**: Track endpoint health and availability

### Example Usage

1. **View HttpFS dashboard**: Navigate to `http://httpfs-host:14000/`

## Configuring Secure Access to Web UIs

All web UIs can be secured with Kerberos authentication:

1. **Enable HTTP Security**:
   ```xml
   <property>
     <name>ozone.security.http.kerberos.enabled</name>
     <value>true</value>
   </property>
   <property>
     <name>ozone.http.filter.initializers</name>
     <value>org.apache.hadoop.security.AuthenticationFilterInitializer</value>
   </property>
   ```

2. **Configure Authentication Type**:
   ```xml
   <property>
     <name>ozone.om.http.auth.type</name>
     <value>kerberos</value>
   </property>
   <property>
     <name>hdds.scm.http.auth.type</name>
     <value>kerberos</value>
   </property>
   <property>
     <name>hdds.datanode.http.auth.type</name>
     <value>kerberos</value>
   </property>
   ```

3. **Configure Service HTTP Principals**:
   ```xml
   <property>
     <name>ozone.om.http.auth.kerberos.principal</name>
     <value>HTTP/_HOST@REALM</value>
   </property>
   <property>
     <name>ozone.om.http.auth.kerberos.keytab</name>
     <value>/etc/security/keytabs/om.http.keytab</value>
   </property>
   ```

## JVM Profiling with Async-Profiler

Ozone web UIs include built-in support for async-profiler to help with performance troubleshooting:

1. **Enable Profiler**:
   ```xml
   <property>
     <name>hdds.profiler.endpoint.enabled</name>
     <value>true</value>
   </property>
   ```

2. **Set Async-Profiler Home** (environment variable or system property):
   ```bash
   export ASYNC_PROFILER_HOME=/path/to/async-profiler
   ```
   
3. **Access Profiler**:
   Visit `http://service-host:port/prof` to generate profiles

4. **Profiling Options**:
   - CPU profiling: `http://host:port/prof`
   - Heap allocation: `http://host:port/prof?event=alloc`
   - Lock contention: `http://host:port/prof?event=lock`
   - Custom duration: `http://host:port/prof?duration=60`

The profiler generates flame graphs that help visualize CPU usage or memory allocations.

## Modifying Log Levels

All Ozone web UIs allow administrators to change log levels dynamically:

1. **View Current Levels**: Access `http://service-host:port/logLevel`
2. **Change a Log Level**: 
   - Use the web form to select logger and level
   - Or use the command line tool: `ozone insight logs --set=org.apache.hadoop.ozone=DEBUG`

## Integration with Prometheus

Ozone services expose metrics in Prometheus format for monitoring:

1. **Access Metrics**: Visit `http://service-host:port/prom`
2. **Configure Prometheus**: Add scrape jobs for each Ozone service
3. **Visualize with Grafana**: Ozone provides example Grafana dashboards
   - Overall Metrics Dashboard
   - Object Metrics Dashboard
   - RPC Metrics Dashboard

## Command Line Access to Web UI Endpoints

The `ozone insight` command provides CLI access to web UI endpoints:

```bash
# View all available metrics
ozone insight metrics list

# View metrics for a specific component
ozone insight metrics --service=scm

# Check configuration
ozone insight config --service=om

# Change log levels
ozone insight logs --service=om --set=org.apache.hadoop.ozone=DEBUG
```

## Troubleshooting Web UI Issues

Common issues and solutions:

1. **Web UI not accessible**
   - Check if the service is running
   - Verify port availability and firewall settings
   - Check for bind errors in service logs

2. **Authentication failures**
   - Ensure Kerberos tickets are valid (`kinit`)
   - Verify HTTP principal and keytab configuration
   - Check for SPNEGO support in your browser

3. **Missing metrics**
   - Check if JMX is enabled
   - Verify service health and logs
   - Ensure metrics collection is properly configured

4. **High latency in web UI**
   - Check for resource constraints (CPU, memory)
   - Review thread pool sizes in configuration
   - Check network connectivity between components

## Best Practices

1. **Security**: Always secure web UIs with Kerberos authentication in production
2. **Monitoring**: Set up Prometheus and Grafana for continuous monitoring
3. **Access Control**: Use `ozone.administrators` and `ozone.recon.administrators` to restrict admin access
4. **Regular Checks**: Periodically review logs and metrics for potential issues
5. **Backup Checkpoints**: Use the checkpoint endpoints to back up metadata regularly

By effectively using these web UIs, administrators can gain valuable insights into their Ozone cluster's health, performance, and configuration.