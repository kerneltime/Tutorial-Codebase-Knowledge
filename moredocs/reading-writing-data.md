---
sidebar_label: Reading and Writing Data
---

# Reading and Writing Data in Ozone

Apache Ozone provides multiple interfaces for reading and writing data, catering to different use cases and client preferences. This guide explains how to use the three primary interfaces within a Docker environment:

1. **Ozone Shell (ozone sh)** - The native command-line interface
2. **OFS (Ozone File System)** - Hadoop-compatible file system interface
3. **S3 API** - Amazon S3 compatible REST interface

All examples assume you already have a running Ozone cluster using Docker Compose as described in the [Docker Installation Guide](docker_installation.md).

## Interface Comparison

| Interface | Strengths | Use Cases |
|-----------|-----------|-----------|
| **Ozone Shell** | - Full feature access<br>- Advanced operations<br>- Detailed metadata | - Administrative tasks<br>- Bucket/volume management<br>- Quota/ACL management |
| **OFS** | - Familiar HDFS-like commands<br>- Works with existing Hadoop applications<br>- Full cluster view | - Hadoop ecosystem integration<br>- Applications that need filesystem semantics |
| **S3 API** | - Industry standard<br>- Works with existing S3 clients<br>- Language-independent | - Web applications<br>- Multi-language environments<br>- Existing S3 applications |

## 1. Using Ozone Shell (ozone sh)

The Ozone Shell provides direct access to all Ozone features through a command-line interface. All commands follow the pattern:

```bash
ozone sh <object-type> <action> <path>
```

Where `<object-type>` is `volume`, `bucket`, or `key`.

### Accessing the Ozone Shell

To use the Ozone Shell in your Docker environment:

```bash
docker compose exec om bash
```

This gives you a shell inside the Ozone Manager (OM) container where you can run the `ozone sh` commands.

### Working with Volumes

Volumes are the top-level namespace in Ozone, conceptually similar to file system mounts.

```bash
# Create a volume
ozone sh volume create /vol1

# List all volumes
ozone sh volume list /

# Get volume details
ozone sh volume info /vol1

# Delete a volume (must be empty)
ozone sh volume delete /vol1

# Delete a volume recursively (deletes all contained buckets and keys)
ozone sh volume delete -r /vol1
```

### Working with Buckets

Buckets are containers for keys (objects) within volumes.

```bash
# Create a bucket
ozone sh bucket create /vol1/bucket1

# List all buckets in a volume
ozone sh bucket list /vol1

# Get bucket details
ozone sh bucket info /vol1/bucket1

# Delete a bucket (must be empty)
ozone sh bucket delete /vol1/bucket1

# Delete a bucket recursively
ozone sh bucket delete -r /vol1/bucket1
```

### Working with Keys (Objects)

Keys are the actual data objects stored in Ozone.

```bash
# Create a test file
echo "Hello Ozone" > test.txt

# Upload a file (put source to destination)
ozone sh key put /vol1/bucket1/test.txt test.txt

# Upload with specific replication type
ozone sh key put -t RATIS /vol1/bucket1/key1_RATIS /opt/hadoop/NOTICE.txt

# Download a file
ozone sh key get /vol1/bucket1/test.txt /tmp/

# Force overwrite when downloading
ozone sh key get --force /vol1/bucket1/test.txt /tmp/test.txt

# Get key information
ozone sh key info /vol1/bucket1/test.txt

# List keys in a bucket
ozone sh key list /vol1/bucket1

# Copy a key within a bucket
ozone sh key cp /vol1/bucket1 test.txt test.txt-copy

# Rename a key
ozone sh key rename /vol1/bucket1 test.txt test.txt-renamed

# Delete a key
ozone sh key delete /vol1/bucket1/test.txt

# In FSO bucket mode, deleted keys move to trash at /<volume>/<bucket>/.Trash/<user>/
# In OBS bucket mode, deleted keys are permanently removed
```

## 2. Using OFS (Ozone File System)

OFS provides a Hadoop-compatible file system interface, making it seamless to use with applications that are designed to work with HDFS.

### OFS Configuration

To use OFS, first make sure the filesystem client is on the classpath. In the Docker environment, this is already set up for you.

```bash
# Inside the OM container
docker compose exec om bash

# Configure HADOOP_CLASSPATH if needed (usually not required in the Docker setup)
export HADOOP_CLASSPATH=/opt/ozone/share/ozone/lib/ozone-filesystem-hadoop3-*.jar:$HADOOP_CLASSPATH
```

### Setting up /tmp Directory (One-time Setup)

For applications that use temporary directories, set up the `/tmp` mount point:

```bash
# Admin setup (once per cluster)
ozone sh volume create tmp
ozone sh volume setacl tmp -al world::a

# User setup (once per user)
ozone fs -mkdir /tmp
```

### Basic OFS Operations

OFS uses the standard Hadoop filesystem commands (`hdfs dfs` or `ozone fs`).

```bash
# Create volume and bucket (using filesystem semantics)
ozone fs -mkdir /vol1
ozone fs -mkdir /vol1/bucket1

# Upload a file
echo "Hello from OFS" > local_test.txt
ozone fs -put local_test.txt /vol1/bucket1/

# Copy from local with explicit destination path
ozone fs -copyFromLocal NOTICE.txt /vol1/bucket1/NOTICE.txt

# List files in a bucket
ozone fs -ls /vol1/bucket1/

# List recursively
ozone fs -ls -R /vol1/

# Download a file
ozone fs -get /vol1/bucket1/local_test.txt ./downloaded_file.txt

# Display file contents
ozone fs -cat /vol1/bucket1/local_test.txt

# Move a file (rename)
ozone fs -mv /vol1/bucket1/NOTICE.txt /vol1/bucket1/MOVED.txt

# Copy a file within the filesystem
ozone fs -cp /vol1/bucket1/MOVED.txt /vol1/bucket1/COPY.txt

# Delete a file (moves to trash if enabled)
ozone fs -rm /vol1/bucket1/COPY.txt

# Delete a file and skip trash
ozone fs -rm -skipTrash /vol1/bucket1/local_test.txt

# Create an empty file
ozone fs -touch /vol1/bucket1/empty_file.txt
```

### Advanced OFS Operations

```bash
# Copy files within the filesystem
ozone fs -cp /vol1/bucket1/file1 /vol1/bucket1/file2

# Move files
ozone fs -mv /vol1/bucket1/file1 /vol1/bucket1/file2 

# Get file checksum
ozone fs -checksum /vol1/bucket1/file1

# Create empty file
ozone fs -touchz /vol1/bucket1/empty_file

# Set replication factor for a file
ozone fs -setrep -w 3 /vol1/bucket1/important_file

# Enable trash (in core-site.xml)
# <property>
#   <name>fs.trash.interval</name>
#   <value>10</value>
# </property>
# <property>
#   <name>fs.trash.classname</name>
#   <value>org.apache.hadoop.ozone.om.TrashPolicyOzone</value>
# </property>
```

## 3. Using S3 API

The S3 API provides compatibility with applications designed to work with Amazon S3. It's accessible from both inside and outside the Docker containers.

### S3 Credentials

In non-secure mode, you can use any values for credentials. For a secure setup, obtain the credentials from Ozone:

```bash
# Inside the OM container
docker compose exec om bash

# Generate S3 credentials
# In non-secure mode, any values work:
export AWS_ACCESS_KEY_ID=testuser
export AWS_SECRET_ACCESS_KEY=testuser-secret

# In secure mode, use this command:
# ozone s3 getsecret
```

### Using AWS CLI

The AWS CLI can be used from outside the Docker containers to interact with Ozone via the S3 Gateway:

```bash
# Set environment variables
export AWS_ACCESS_KEY_ID=testuser
export AWS_SECRET_ACCESS_KEY=testuser-secret

# Create a bucket
aws s3api --endpoint http://localhost:9878/ create-bucket --bucket=bucket1

# List buckets
aws s3api --endpoint http://localhost:9878/ list-buckets

# Upload a file
echo "Hello S3" > s3_test.txt
aws s3 --endpoint http://localhost:9878 cp s3_test.txt s3://bucket1/

# List objects in a bucket
aws s3 --endpoint http://localhost:9878 ls s3://bucket1/

# Download a file
aws s3 --endpoint http://localhost:9878 cp s3://bucket1/s3_test.txt ./downloaded_s3.txt

# Delete an object
aws s3 --endpoint http://localhost:9878 rm s3://bucket1/s3_test.txt

# Delete a bucket
aws s3api --endpoint http://localhost:9878/ delete-bucket --bucket=bucket1
```

## Cross-Interface Operations

Apache Ozone is designed as a multi-protocol aware storage system, allowing you to access the same data through different interfaces. This unified approach gives you flexibility in choosing the right interface for each task while maintaining a single source of truth for your data.

### Protocol Compatibility Matrix

| Interface | Data Model | Path Convention | Unique Features | Cross-Interface Compatibility |
|-----------|------------|-----------------|-----------------|------------------------------|
| **Ozone Shell** | Native | `/volume/bucket/key` | Full admin capabilities | Can access all data |
| **OFS** | HDFS-like global view | `ofs://om-host/volume/bucket/key` | Global namespace, HDFS compatibility | Can access all data |
| **O3FS** | HDFS-like bucket view | `o3fs://bucket.volume/key` | Legacy application support | Limited to single bucket |
| **S3** | S3-compatible | `s3://bucket/key` | Industry standard | Limited to `/s3v` volume or linked buckets |

### Namespace Mapping Between Interfaces

The same data can be accessed through different path conventions:

| Data Location | Ozone Shell | OFS | O3FS | S3 |
|---------------|-------------|----|------|---|
| Vol1/bucket1/file.txt | `/vol1/bucket1/file.txt` | `ofs://om/vol1/bucket1/file.txt` | `o3fs://bucket1.vol1/file.txt` | Need bucket link |
| s3v/bucket2/file.txt | `/s3v/bucket2/file.txt` | `ofs://om/s3v/bucket2/file.txt` | `o3fs://bucket2.s3v/file.txt` | `s3://bucket2/file.txt` |

### Accessing S3 Data Through OFS or Ozone Shell

S3 buckets are stored under the `/s3v` volume in Ozone's namespace. This allows you to access objects created through the S3 interface using OFS or Ozone Shell:

```bash
# Create a bucket and upload a file using S3
aws s3api --endpoint http://localhost:9878/ create-bucket --bucket=shared-bucket
echo "Test data" > test.txt
aws s3 --endpoint http://localhost:9878 cp test.txt s3://shared-bucket/

# Access the same data from OFS
ozone fs -ls /s3v/shared-bucket/
ozone fs -cat /s3v/shared-bucket/test.txt

# Access the same data using Ozone Shell
ozone sh key list /s3v/shared-bucket
ozone sh key get /s3v/shared-bucket/test.txt /tmp/test-copy.txt
```

### Exposing Ozone Buckets via S3

You can make any Ozone bucket accessible through the S3 interface using bucket links, regardless of which volume it belongs to:

```bash
# Create a volume and bucket using Ozone Shell
ozone sh volume create /vol1
ozone sh bucket create /vol1/bucket1

# Create an S3-accessible symbolic link
ozone sh bucket link /vol1/bucket1 /s3v/shared-bucket

# Upload data using Ozone Shell
echo "Hello from Ozone Shell" > hello.txt
ozone sh key put /vol1/bucket1/hello.txt hello.txt

# Now access the bucket data via S3
aws s3 --endpoint http://localhost:9878 ls s3://shared-bucket/
aws s3 --endpoint http://localhost:9878 cp s3://shared-bucket/hello.txt downloaded.txt
```

### O3FS vs OFS Interface

Ozone provides two filesystem interfaces that serve different use cases:

1. **O3FS (o3fs://)**: Bucket-specific view, suitable for applications focused on a single bucket
2. **OFS (ofs://)**: Global view of all volumes and buckets, provides a unified namespace

```bash
# Access using O3FS (bucket-specific)
hdfs dfs -ls o3fs://bucket1.vol1/

# Access the same data using OFS (global namespace)
hdfs dfs -ls ofs://om-host/vol1/bucket1/
```

### Bucket Layouts and Their Impact on Interfaces

Ozone supports two bucket layout types that affect how data can be accessed:

1. **FILE_SYSTEM_OPTIMIZED (FSO)**: 
   - Optimized for hierarchical filesystem operations
   - Better performance for directory operations
   - Supports file rename within the bucket
   - Deleted keys go to trash when deleted through OFS/O3FS
   - Recommended for Hadoop workloads

2. **OBJECT_STORE (OBS/legacy)**:
   - Optimized for object operations
   - Flat namespace internally
   - Better performance for direct key access
   - No trash support (deleted keys are permanently removed)
   - Recommended for S3 workloads

```bash
# Create buckets with different layouts
ozone sh bucket create /vol1/fsobucket --layout FILE_SYSTEM_OPTIMIZED
ozone sh bucket create /vol1/obsbucket --layout OBJECT_STORE

# OFS operations work better with FSO buckets
ozone fs -mkdir -p /vol1/fsobucket/dir1/dir2
ozone fs -put file.txt /vol1/fsobucket/dir1/
ozone fs -mv /vol1/fsobucket/dir1/file.txt /vol1/fsobucket/dir1/renamed.txt  # Works with FSO

# S3 operations work with both types
aws s3 --endpoint http://localhost:9878 cp file.txt s3://obsbucket/prefix/file.txt  # Better with OBS
```

### Protocol-Specific Behaviors

While data is accessible across interfaces, be aware of these protocol-specific behaviors:

1. **Metadata Handling**:
   - S3 objects have S3-specific metadata (ETag, Content-Type, etc.)
   - OFS/O3FS files have HDFS-like attributes (permissions, owner/group, etc.)

2. **Key/Filename Constraints**:
   - S3 keys can contain characters that might be problematic in filesystem paths
   - OFS/O3FS have stricter path component requirements

3. **Deletion Behavior**:
   - S3 or Ozone Shell deletions are permanent
   - OFS/O3FS deletions move files to trash in FSO buckets when trash is enabled
