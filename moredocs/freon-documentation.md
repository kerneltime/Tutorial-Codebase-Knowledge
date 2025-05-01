# Apache Ozone Freon Command Line Documentation

## Overview

Freon is a load generator and performance testing tool for Apache Ozone. It provides various subcommands to test different components of the Ozone ecosystem, including key generation, metadata operations, datanode operations, and more. Freon is designed to help users benchmark, stress test, and validate their Ozone deployments.

## Basic Usage

```bash
ozone freon [options] [subcommand] [subcommand-options]
```

## Global Options

These options apply to all Freon subcommands:

| Option | Description | Default |
|--------|-------------|---------|
| `--server` | Enable internal HTTP server to provide metrics and profile endpoint | `false` |
| `-h, --help` | Show help message | - |
| `--version` | Show version information | - |

## Common Subcommand Options

Most Freon subcommands support these common options:

| Option | Description | Default |
|--------|-------------|---------|
| `-n, --number-of-tests` | Number of objects to generate | `1000` |
| `-t, --threads` | Number of threads to use for execution | `10` |
| `--duration` | Duration to run the test (e.g., '30s', '5m', '1h', '7d') | - |
| `-f, --fail-at-end` | Continue execution even if there are failures | `false` |
| `-p, --prefix` | Unique identifier for the test execution (used as prefix for generated object names) | Random string |
| `--verbose` | Show more verbose output including all command line options | `false` |

## Subcommands

### RandomKeyGenerator (rk, randomkeys)

Generates volumes, buckets, and keys in Ozone to test write and read performance.

```bash
ozone freon randomkeys [options]
```

#### Options

| Option | Description | Default |
|--------|-------------|---------|
| `--num-of-volumes` | Number of volumes to create | `10` |
| `--num-of-buckets` | Number of buckets to create per volume | `1000` |
| `--num-of-keys` | Number of keys to create per bucket | `500000` |
| `--key-size` | Size of each key in bytes | `10KB` |
| `--validate-writes` | Validate keys after writing | `false` |
| `--num-of-validate-threads` | Number of threads for key validation | `1` |
| `--buffer-size` | Buffer size for writing | `4096` |
| `--json` | Directory where JSON report is created | - |
| `--om-service-id` | OM Service ID | - |
| `--clean-objects` | Clean generated volumes, buckets, and keys after test | `false` |

#### Example

```bash
# Generate 10,000 keys of 1MB each using 20 threads
ozone freon randomkeys -t 20 -n 10000 --key-size=1MB

# Generate keys and validate writes
ozone freon rk -t 20 -n 1000 --validate-writes
```

### OmMetadataGenerator (ommg, om-metadata-generator)

Tests Ozone Manager (OM) performance with various metadata operations.

```bash
ozone freon ommg --operation <OPERATION> [options]
```

#### Operations

- `CREATE_FILE`: Create files
- `CREATE_STREAM_FILE`: Create stream files
- `LOOKUP_FILE`: Look up files
- `READ_FILE`: Read files
- `LIST_STATUS`: List file status
- `LIST_STATUS_LIGHT`: List file status (light version)
- `CREATE_KEY`: Create keys
- `CREATE_STREAM_KEY`: Create stream keys
- `LOOKUP_KEY`: Look up keys
- `GET_KEYINFO`: Get key info
- `HEAD_KEY`: Head key operation
- `READ_KEY`: Read keys
- `LIST_KEYS`: List keys
- `LIST_KEYS_LIGHT`: List keys (light version)
- `INFO_BUCKET`: Get bucket info
- `INFO_VOLUME`: Get volume info
- `MIXED`: Run mixed operations

#### Options

| Option | Description | Default |
|--------|-------------|---------|
| `-v, --volume` | Volume name (created if missing) | `vol1` |
| `-b, --bucket` | Bucket name (created if missing) | `bucket1` |
| `-s, --size` | Size of file/key to create | `0` |
| `--buffer` | Buffer size for generating content | `4096` |
| `--batch-size` | Number of keys/files per LIST operation | `1000` |
| `--random` | Enable random read/write operations | `false` |
| `--ops` | Operations list for MIXED mode | - |
| `--opsnum` | Number of threads per operation for MIXED mode | - |
| `--ophelp` | Print operation help | `false` |
| `--om-service-id` | OM Service ID | - |

#### Examples

```bash
# Create 25,000 keys with 20 threads for 3 minutes
ozone freon ommg --operation CREATE_KEY -t 20 -n 25000 --duration 3m

# Read 10,000 keys
ozone freon ommg --operation READ_KEY -n 10000

# List 1,000 keys per request with 20 threads for 3 minutes
ozone freon ommg --operation LIST_KEYS -t 20 --batch-size 1000 --duration 3m

# Mixed operations: 5 threads creating files, 4 threads looking up files, 1 thread listing status
ozone freon ommg --operation MIXED --ops CREATE_FILE,LOOKUP_FILE,LIST_STATUS --opsnum 5,4,1 -t 10 -n 1000
```

### DatanodeChunkGenerator (dcg, datanode-chunk-generator)

Tests datanode performance by writing chunks directly using the XCeiver interface.

```bash
ozone freon dcg [options]
```

#### Options

| Option | Description | Default |
|--------|-------------|---------|
| `-a, --async` | Use async operation | `false` |
| `-s, --size` | Size of generated chunks in bytes | `1024` |
| `-l, --pipeline` | Pipeline ID to use | First RATIS/THREE pipeline |
| `-d, --datanodes` | Datanodes to use | - |

#### Example

```bash
# Generate chunks of 4KB using 20 threads
ozone freon dcg -t 20 -s 4096 -n 10000

# Use async operations
ozone freon dcg -t 20 -n 5000 --async
```

### DNRPCLoadGenerator (dnrpc)

Tests DataNode RPC performance.

```bash
ozone freon dnrpc [options]
```

#### Example

```bash
ozone freon dnrpc -t 10 -n 1000
```

### HadoopDirTreeGenerator (dirtree)

Generates directory trees in Hadoop-compatible filesystems.

```bash
ozone freon dirtree [options]
```

#### Example

```bash
ozone freon dirtree -t 5 -n 1000 --path o3fs://bucket1.vol1/test
```

### HadoopNestedDirGenerator (nested)

Generates nested directories in Hadoop-compatible filesystems.

```bash
ozone freon nested [options]
```

### HsyncGenerator (hsync-generator)

Tests hsync performance.

```bash
ozone freon hsync-generator [options]
```

#### Example

```bash
ozone freon hsync-generator -t 5 --writes-per-transaction=32 --bytes-per-write=8 -n 1000000
```

### S3KeyGenerator (s3g)

Generates S3 keys to test S3 gateway performance.

```bash
ozone freon s3g [options]
```

### S3BucketGenerator (s3bg)

Generates S3 buckets to test S3 gateway performance.

```bash
ozone freon s3bg [options]
```

### DatanodeSimulator (simulate-datanode)

Simulates datanode heartbeats to test SCM performance.

```bash
ozone freon simulate-datanode [options]
```

#### Example

```bash
ozone freon simulate-datanode -t 20 -n 5000 -c 40000
```

## Container Generator Commands

### GeneratorDatanode (cgdn)

Generates containers on datanodes.

```bash
ozone freon cgdn [options]
```

### GeneratorScm (cgscm)

Generates container metadata in SCM.

```bash
ozone freon cgscm [options]
```

### GeneratorOm (cgom)

Generates metadata in OM.

```bash
ozone freon cgom [options]
```

## Output and Metrics

Freon provides detailed metrics for each test run, including:

- Total execution time
- Number of successful operations
- Number of failed operations
- Operation throughput
- Latency statistics (mean, deviation, percentiles)

When using the `--server` option, Freon starts an HTTP server that provides:

- Real-time metrics via HTTP endpoint
- JVM profiling information

## Examples of Common Use Cases

### Basic Performance Testing

```bash
# Generate 10,000 keys with 20 threads
ozone freon randomkeys -t 20 -n 10000

# Test OM performance with key creation
ozone freon ommg --operation CREATE_KEY -t 20 -n 10000
```

### Time-based Testing

```bash
# Run key generation test for 30 minutes
ozone freon randomkeys -t 20 --duration 30m

# Run OM metadata operations for 1 hour
ozone freon ommg --operation MIXED --ops CREATE_KEY,READ_KEY --opsnum 10,10 -t 20 --duration 1h
```

### Validation Testing

```bash
# Generate keys and validate the writes
ozone freon randomkeys -t 20 -n 10000 --validate-writes --num-of-validate-threads 5
```

### Datanode Testing

```bash
# Test datanode chunk write performance
ozone freon dcg -t 20 -n 10000 -s 1MB
```

### S3 Gateway Testing

```bash
# Test S3 gateway with key generation
ozone freon s3g -t 10 -n 1000
```

## Best Practices

1. **Start Small**: Begin with a small number of operations to verify your setup before running large-scale tests.

2. **Monitor Resources**: Monitor CPU, memory, and disk usage during tests to identify bottlenecks.

3. **Use Appropriate Thread Count**: The optimal thread count depends on your hardware. Start with a number equal to the number of CPU cores and adjust as needed.

4. **Clean Up After Tests**: Use the `--clean-objects` option with RandomKeyGenerator to clean up test data after completion.

5. **Use Time-based Tests for Stability**: For stability testing, use the `--duration` option to run tests for extended periods.

6. **Validate Writes**: For data integrity testing, use the `--validate-writes` option with RandomKeyGenerator.

7. **Use HTTP Server for Monitoring**: Enable the `--server` option to monitor test progress and metrics via HTTP.

## Troubleshooting

### Common Issues

1. **Out of Memory Errors**: Reduce the number of threads or the size of objects being generated.

2. **Connection Timeouts**: Check network connectivity between Freon client and Ozone cluster.

3. **Permission Denied**: Ensure the user running Freon has appropriate permissions.

4. **Pipeline Not Found**: Ensure that the Ozone cluster is healthy and has active pipelines.

### Debugging Tips

1. Use the `--verbose` option for more detailed output.

2. Check Ozone component logs (OM, SCM, DataNode) for errors.

3. Enable the HTTP server with `--server` to monitor metrics during test execution.

4. For S3 tests, verify S3 gateway configuration and connectivity.
```bash
# Run key generation test for 30 minutes
ozone freon randomkeys -t 20 --duration 30m

# Run OM metadata operations for 1 hour
ozone freon ommg --operation MIXED --ops CREATE_KEY,READ_KEY --opsnum 10,10 -t 20 --duration 1h
```

### Validation Testing

```bash
# Generate keys and validate the writes
ozone freon randomkeys -t 20 -n 10000 --validate-writes --num-of-validate-threads 5
```

### Datanode Testing

```bash
# Test datanode chunk write performance
ozone freon dcg -t 20 -n 10000 -s 1MB
```

### S3 Gateway Testing

```bash
# Test S3 gateway with key generation
ozone freon s3g -t 10 -n 1000
```

## Best Practices

1. **Start Small**: Begin with a small number of operations to verify your setup before running large-scale tests.

2. **Monitor Resources**: Monitor CPU, memory, and disk usage during tests to identify bottlenecks.

3. **Use Appropriate Thread Count**: The optimal thread count depends on your hardware. Start with a number equal to the number of CPU cores and adjust as needed.

4. **Clean Up After Tests**: Use the `--clean-objects` option with RandomKeyGenerator to clean up test data after completion.

5. **Use Time-based Tests for Stability**: For stability testing, use the `--duration` option to run tests for extended periods.

6. **Validate Writes**: For data integrity testing, use the `--validate-writes` option with RandomKeyGenerator.

7. **Use HTTP Server for Monitoring**: Enable the `--server` option to monitor test progress and metrics via HTTP.

## Troubleshooting

### Common Issues

1. **Out of Memory Errors**: Reduce the number of threads or the size of objects being generated.

2. **Connection Timeouts**: Check network connectivity between Freon client and Ozone cluster.

3. **Permission Denied**: Ensure the user running Freon has appropriate permissions.

4. **Pipeline Not Found**: Ensure that the Ozone cluster is healthy and has active pipelines.

### Debugging Tips

1. Use the `--verbose` option for more detailed output.

2. Check Ozone component logs (OM, SCM, DataNode) for errors.

3. Enable the HTTP server with `--server` to monitor metrics during test execution.

4. For S3 tests, verify S3 gateway configuration and connectivity.
