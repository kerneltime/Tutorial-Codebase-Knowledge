---
sidebar_label: Docker
---

# Try Ozone With Docker

Apache Ozone can be quickly deployed using Docker Compose, making it ideal for development, testing, and evaluation purposes. This guide walks you through setting up and configuring a multi-node Ozone cluster using pre-built Docker images.

## Prerequisites

- [Docker Engine](https://docs.docker.com/engine/install/) - Latest stable version
- [Docker Compose](https://docs.docker.com/compose/install/) - Latest stable version (both v1 and v2 are supported)

## Running Ozone

### Obtain the Docker Compose Configuration

First, obtain Ozone's sample Docker Compose configuration:

```bash
# Download the latest Docker Compose configuration file
curl -O https://raw.githubusercontent.com/apache/ozone-docker/refs/heads/latest/docker-compose.yaml
```

### Start the Cluster

Start your Ozone cluster with three Datanodes using the following command:

```bash
docker compose up -d --scale datanode=3
```

This command will:

- Automatically pull required images from Docker Hub
- Create a multi-node cluster with the core Ozone services (SCM, OM, Datanodes, S3 Gateway, Recon)
- Start all components in detached mode

### Verify the Deployment

1. Check the status of your Ozone cluster components:

    ```bash
    docker compose ps
    ```

   You should see output similar to this:

    ```bash
    NAME                IMAGE                      COMMAND                  SERVICE    CREATED          STATUS          PORTS
    docker-datanode-1   apache/ozone:1.4.1-rocky   "/usr/local/bin/dumb…"   datanode   14 seconds ago   Up 13 seconds   0.0.0.0:32958->9864/tcp, :::32958->9864/tcp
    docker-datanode-2   apache/ozone:1.4.1-rocky   "/usr/local/bin/dumb…"   datanode   14 seconds ago   Up 13 seconds   0.0.0.0:32957->9864/tcp, :::32957->9864/tcp
    docker-datanode-3   apache/ozone:1.4.1-rocky   "/usr/local/bin/dumb…"   datanode   14 seconds ago   Up 12 seconds   0.0.0.0:32959->9864/tcp, :::32959->9864/tcp
    docker-om-1         apache/ozone:1.4.1-rocky   "/usr/local/bin/dumb…"   om         14 seconds ago   Up 13 seconds   0.0.0.0:9874->9874/tcp, :::9874->9874/tcp
    docker-recon-1      apache/ozone:1.4.1-rocky   "/usr/local/bin/dumb…"   recon      14 seconds ago   Up 13 seconds   0.0.0.0:9888->9888/tcp, :::9888->9888/tcp
    docker-s3g-1        apache/ozone:1.4.1-rocky   "/usr/local/bin/dumb…"   s3g        14 seconds ago   Up 13 seconds   0.0.0.0:9878->9878/tcp, :::9878->9878/tcp
    docker-scm-1        apache/ozone:1.4.1-rocky   "/usr/local/bin/dumb…"   scm        14 seconds ago   Up 13 seconds   0.0.0.0:9876->9876/tcp, :::9876->9876/tcp
    ```

2. Check the Ozone version:

    ```bash
    docker compose exec om ozone version
    ```

3. Access the web interfaces for monitoring and management:
   - [Ozone Manager UI](http://localhost:9874/) - Manage volumes, buckets, and keys
   - [Storage Container Manager UI](http://localhost:9876/) - View container and datanode status
   - [Recon Server UI](http://localhost:9888/) - Comprehensive monitoring dashboard
   - [S3 Gateway](http://localhost:9878/) - S3-compatible REST endpoint

## Configuration

### Basic Configuration

You can customize your Ozone deployment by modifying the configuration parameters in the `docker-compose.yaml` file:

1. **Common Configurations**: Located under the `x-common-config` section
2. **Service-Specific Settings**: Found under the `environment` section of individual services

To apply configuration changes, use environment variables with the prefix `OZONE-SITE.XML_` followed by the configuration key. For example:

```yaml
x-common-config:
  ...
  OZONE-SITE.XML_ozone.recon.http-address: 0.0.0.0:9090
  OZONE-SITE.XML_ozone.server.default.replication: 3
```

### Advanced Configurations

#### Replication Factor

By default, a single-node deployment has a replication factor of 1. For multi-node deployments, you may want to increase this:

```yaml
x-common-config:
  ...
  OZONE-SITE.XML_ozone.server.default.replication: 3
```

#### Adding Wait Time for Service Dependencies

If you need services to wait for other dependencies, use the `WAITFOR` environment variable:

```yaml
services:
  s3g:
    ...
    environment:
      WAITFOR: om:9874
      WAITFOR_TIMEOUT: 120
```

#### Security Configuration

For secure deployments with Kerberos, additional settings are available:

```yaml
x-common-config:
  ...
  KERBEROS_ENABLED: "true"
  KERBEROS_KEYTABS: "om datanode s3g"
```

## Using Ozone with S3 API

Ozone provides S3 API compatibility through the S3 Gateway service, which is accessible on port 9878 by default:

1. Create a bucket via the S3 API:

   ```bash
   # Using AWS CLI
   aws s3api --endpoint http://localhost:9878/ create-bucket --bucket=my-bucket
   ```

2. Upload a file to your bucket:

   ```bash
   # Create a test file
   echo "Hello Ozone" > test.txt
   
   # Upload using AWS CLI
   aws s3 --endpoint http://localhost:9878 cp test.txt s3://my-bucket/
   ```

3. List objects in your bucket:

   ```bash
   aws s3 --endpoint http://localhost:9878 ls s3://my-bucket/
   ```

## Advanced Deployment Options

### Single Container Deployment

For the simplest deployment, you can run all Ozone services in a single container:

```bash
docker run -p 9878:9878 -p 9876:9876 -p 9874:9874 apache/ozone
```

### High Availability Deployment

For production-like environments, you can use the HA configuration:

```bash
# Download HA configuration
curl -O https://raw.githubusercontent.com/apache/ozone/master/hadoop-ozone/dist/src/main/compose/ozone-ha/docker-compose.yaml

# Start HA cluster
docker compose -f docker-compose.yaml up -d
```

## Troubleshooting

### Common Issues

1. **Container startup failures**: Check logs with `docker compose logs [service]`
2. **Connection issues**: Ensure ports are not blocked by firewalls
3. **Low disk space**: Verify available storage for container volumes

### Viewing Logs

To view logs for a specific service:

```bash
docker compose logs -f om  # View Ozone Manager logs
docker compose logs -f scm  # View Storage Container Manager logs
docker compose logs -f datanode  # View Datanode logs
```

## Next Steps

Now that your Ozone cluster is up and running, you can:

1. Enter any container to run commands directly:

   ```bash
   docker compose exec om bash
   ```

2. Learn how to [read and write data](reading-writing-data.md) into Ozone.

3. Explore high availability setups, secure deployments, and integration with other systems.

4. For production deployments, refer to the [Configuring Ozone For Production](/docs/quick-start/installation/docker) guide.