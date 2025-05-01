# Configuring Kerberos Security for Apache Ozone

This document provides a comprehensive guide to setting up Kerberos authentication for Apache Ozone in both bare metal and Docker-based environments.

## Prerequisites

- A functioning Kerberos KDC (Key Distribution Center)
- Basic knowledge of Kerberos concepts (principals, keytabs)
- Properly configured DNS (hostname resolution is critical for Kerberos)

## Core Configuration Steps

### 1. Enable Security in Ozone

Edit `ozone-site.xml` to enable security:

```xml
<!-- Basic Security Settings -->
<property>
  <name>ozone.security.enabled</name>
  <value>true</value>
</property>

<!-- Enable Kerberos authentication -->
<property>
  <name>hadoop.security.authentication</name>
  <value>kerberos</value>
</property>
```

### 2. Create Kerberos Principals and Keytabs

For each Ozone service, create the following principals:

| Service | Principal Format | Keytab Location |
|---------|-----------------|-----------------|
| SCM | `scm/_HOST@REALM` | `/etc/security/keytabs/scm.keytab` |
| OM | `om/_HOST@REALM` | `/etc/security/keytabs/om.keytab` |
| DataNode | `dn/_HOST@REALM` | `/etc/security/keytabs/dn.keytab` |
| S3 Gateway | `s3g/_HOST@REALM` | `/etc/security/keytabs/s3g.keytab` |
| Recon | `recon/_HOST@REALM` | `/etc/security/keytabs/recon.keytab` |

Also create HTTP principals for each service's web interface:
- `HTTP/_HOST@REALM`

Example kadmin commands:
```bash
# For OM principal and keytab
kadmin -q "addprinc -randkey om/om.example.com@EXAMPLE.COM"
kadmin -q "ktadd -k /etc/security/keytabs/om.keytab om/om.example.com@EXAMPLE.COM"

# For HTTP principal (for web UI)
kadmin -q "addprinc -randkey HTTP/om.example.com@EXAMPLE.COM"
kadmin -q "ktadd -k /etc/security/keytabs/om.http.keytab HTTP/om.example.com@EXAMPLE.COM"
```

> **Note:** Use the `_HOST` placeholder in configuration to automatically substitute the correct hostname.

### 3. Configure Service Principals in ozone-site.xml

```xml
<!-- SCM Kerberos configuration -->
<property>
  <name>hdds.scm.kerberos.principal</name>
  <value>scm/_HOST@EXAMPLE.COM</value>
</property>
<property>
  <name>hdds.scm.kerberos.keytab.file</name>
  <value>/etc/security/keytabs/scm.keytab</value>
</property>

<!-- OM Kerberos configuration -->
<property>
  <name>ozone.om.kerberos.principal</name>
  <value>om/_HOST@EXAMPLE.COM</value>
</property>
<property>
  <name>ozone.om.kerberos.keytab.file</name>
  <value>/etc/security/keytabs/om.keytab</value>
</property>

<!-- DataNode Kerberos configuration -->
<property>
  <name>hdds.datanode.kerberos.principal</name>
  <value>dn/_HOST@EXAMPLE.COM</value>
</property>
<property>
  <name>hdds.datanode.kerberos.keytab.file</name>
  <value>/etc/security/keytabs/dn.keytab</value>
</property>
```

### 4. Configure HTTP Security (Web UI)

```xml
<!-- Enable HTTP Security -->
<property>
  <name>ozone.security.http.kerberos.enabled</name>
  <value>true</value>
</property>
<property>
  <name>ozone.http.filter.initializers</name>
  <value>org.apache.hadoop.security.AuthenticationFilterInitializer</value>
</property>

<!-- HTTP Authentication Type -->
<property>
  <name>hdds.scm.http.auth.type</name>
  <value>kerberos</value>
</property>
<property>
  <name>ozone.om.http.auth.type</name>
  <value>kerberos</value>
</property>
<property>
  <name>hdds.datanode.http.auth.type</name>
  <value>kerberos</value>
</property>

<!-- HTTP Principals and Keytabs -->
<property>
  <name>hdds.scm.http.auth.kerberos.principal</name>
  <value>HTTP/_HOST@EXAMPLE.COM</value>
</property>
<property>
  <name>hdds.scm.http.auth.kerberos.keytab</name>
  <value>/etc/security/keytabs/scm.http.keytab</value>
</property>
<property>
  <name>ozone.om.http.auth.kerberos.principal</name>
  <value>HTTP/_HOST@EXAMPLE.COM</value>
</property>
<property>
  <name>ozone.om.http.auth.kerberos.keytab</name>
  <value>/etc/security/keytabs/om.http.keytab</value>
</property>
```

### 5. Configure core-site.xml

```xml
<property>
  <name>hadoop.security.authentication</name>
  <value>kerberos</value>
</property>
<property>
  <name>hadoop.security.auth_to_local</name>
  <value>DEFAULT</value>
</property>
```

## Client Authentication

Clients must authenticate using Kerberos to access Ozone:

```bash
# Authentication with username/password
kinit username

# Authentication with keytab
kinit -kt /path/to/user.keytab username@REALM.COM
```

After authenticating, use Ozone commands normally:

```bash
ozone sh volume create /volume1
ozone sh bucket create /volume1/bucket1
```

## Docker-Based Environment Setup

For testing or development environments, you can use Docker to set up a Kerberos-secured Ozone cluster:

### 1. Directory Structure

```
ozonesecure/
  ├── docker-compose.yaml    # Defines all services including KDC
  ├── docker-config/         # Contains service-specific configurations
  └── krb5.conf              # Kerberos client configuration
```

### 2. Example docker-compose.yaml

```yaml
version: "3"
services:
  kdc:
    image: apache/ozone-runner:${OZONE_RUNNER_VERSION:-latest}
    hostname: kdc
    volumes:
      - ./krb5.conf:/etc/krb5.conf
      - ./docker-config:/etc/security
    command: ["kdc"]

  om:
    image: apache/ozone:${OZONE_VERSION:-latest}
    hostname: om
    environment:
      OZONE-SITE.XML_ozone.security.enabled: "true"
      OZONE-SITE.XML_hadoop.security.authentication: "kerberos"
      OZONE-SITE.XML_ozone.om.kerberos.principal: "om/_HOST@EXAMPLE.COM"
      OZONE-SITE.XML_ozone.om.kerberos.keytab.file: "/etc/security/keytabs/om.keytab"
    volumes:
      - ./krb5.conf:/etc/krb5.conf
      - ./docker-config:/etc/security
```

### 3. KDC Initialization Script

Create a script (`init-kdc.sh`) to set up Kerberos principals:

```bash
#!/bin/bash

# Create principals for each service
export_keytab() {
  local principal=$1
  local keytab_file=$2
  
  kadmin.local -q "addprinc -randkey $principal"
  kadmin.local -q "ktadd -k $keytab_file $principal"
  chmod 400 $keytab_file
}

# Create admin principal
kadmin.local -q "addprinc -pw admin admin/admin"

# Create service principals
export_keytab "scm/scm@EXAMPLE.COM" "/etc/security/keytabs/scm.keytab"
export_keytab "HTTP/scm@EXAMPLE.COM" "/etc/security/keytabs/scm.http.keytab"

export_keytab "om/om@EXAMPLE.COM" "/etc/security/keytabs/om.keytab"
export_keytab "HTTP/om@EXAMPLE.COM" "/etc/security/keytabs/om.http.keytab"

export_keytab "dn/dn1@EXAMPLE.COM" "/etc/security/keytabs/dn1.keytab"
export_keytab "HTTP/dn1@EXAMPLE.COM" "/etc/security/keytabs/dn1.http.keytab"

# Create test user
export_keytab "testuser@EXAMPLE.COM" "/etc/security/keytabs/testuser.keytab"
```

## S3 Gateway with Kerberos

For S3 access with Kerberos authentication:

### 1. Configure S3 Gateway principals

```xml
<property>
  <name>ozone.s3g.kerberos.principal</name>
  <value>s3g/_HOST@EXAMPLE.COM</value>
</property>
<property>
  <name>ozone.s3g.kerberos.keytab.file</name>
  <value>/etc/security/keytabs/s3g.keytab</value>
</property>
<property>
  <name>ozone.s3g.http.auth.kerberos.principal</name>
  <value>HTTP/_HOST@EXAMPLE.COM</value>
</property>
<property>
  <name>ozone.s3g.http.auth.kerberos.keytab</name>
  <value>/etc/security/keytabs/s3g.http.keytab</value>
</property>
```

### 2. Obtain S3 credentials

After authenticating with Kerberos, get AWS-compatible S3 credentials:

```bash
# Authenticate first
kinit -kt /etc/security/keytabs/testuser.keytab testuser@EXAMPLE.COM

# Get S3 credentials
ozone s3 getsecret
```

This returns AWS-compatible access key and secret key that can be used with S3 clients.

## Token Management

Ozone uses tokens to reduce the load on the Kerberos KDC:

### 1. Get a delegation token

```bash
# Authenticate first
kinit

# Get a delegation token
ozone sh token get -t token_file
```

### 2. Renew a token

```bash
ozone sh token renew -t token_file
```

### 3. Cancel a token

```bash
ozone sh token cancel -t token_file
```

## Integration with Applications

### Hadoop Integration

Configure `core-site.xml` in Hadoop to use Ozone with Kerberos:

```xml
<property>
  <name>fs.defaultFS</name>
  <value>ofs://om:9862</value>
</property>
<property>
  <name>hadoop.security.authentication</name>
  <value>kerberos</value>
</property>
```

### Spark Integration

For Spark with Kerberos-secured Ozone:

1. Authenticate using Kerberos
2. Get a delegation token
3. Configure Spark to use the token:

```bash
spark-submit --conf spark.hadoop.fs.ofs.impl=org.apache.hadoop.fs.ozone.OzoneFileSystem \
  --conf spark.hadoop.fs.defaultFS=ofs://om:9862 \
  --conf spark.hadoop.hadoop.security.authentication=kerberos \
  --files /etc/hadoop/conf/core-site.xml,/etc/hadoop/conf/ozone-site.xml \
  --conf spark.hadoop.HADOOP_TOKEN_FILE_LOCATION=/path/to/token_file \
  your_application.jar
```

## Troubleshooting

Common issues and solutions:

1. **GSS initiate failed errors**
   - Check that DNS and reverse DNS are working correctly
   - Verify that the time is synchronized between all hosts
   - Make sure principals exactly match the hostnames

2. **Keytab permission issues**
   - Ensure keytabs are owned by the service user and have permission 400 or 600

3. **Authentication failures**
   - Check KDC logs for failed authentication attempts
   - Verify that the principal names in configuration match the ones in keytabs

4. **Service fails to start with Kerberos enabled**
   - Check service logs for Kerberos-related exceptions
   - Verify that all required principals and keytabs exist
   - Ensure `_HOST` substitution is working as expected

5. **Cannot access web UI**
   - Ensure HTTP principals and keytabs are correctly configured
   - Verify browser is configured for Kerberos SPNEGO authentication

## Security Best Practices

1. Use strong passwords for Kerberos admin and user principals
2. Restrict access to keytab files (chmod 400)
3. Use different principals for different services
4. Set reasonable token lifetimes
5. Use firewalls to restrict access to the KDC
6. Configure proper TLS/SSL for web UIs and S3 Gateway

---

This guide covers the essential aspects of configuring Kerberos security for Apache Ozone, suitable for both bare metal and Docker-based environments. For more detailed information, refer to the Apache Ozone documentation.