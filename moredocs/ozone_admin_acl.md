## ozone admin acl

Manages Access Control Lists (ACLs) for Ozone resources.

### Subcommands

#### getacl

Displays the ACLs for a given Ozone resource.

```bash
ozone admin acl getacl <resource_type> <resource_name> [options]
```

##### Resource Types

- `volume`:  Applies to a volume.
- `bucket`: Applies to a bucket.
- `key`: Applies to a key.

##### Options (for bucket resource type)

- `--source`: Display source bucket ACLs if `--source=true`. Only applicable to bucket resource type.

##### Example

```bash
ozone admin acl getacl volume myvolume
ozone admin acl getacl bucket mybucket
ozone admin acl getacl bucket mylinkbucket --source
ozone admin acl getacl key o3fs://mybucket/mykey
```

#### addacl

Adds ACLs to an Ozone resource.

```bash
ozone admin acl addacl <resource_type> <resource_name> --acls <acl_list>
```

##### Resource Types

- `volume`:  Applies to a volume.
- `bucket`: Applies to a bucket.
- `key`: Applies to a key.

##### Options

- `--acls <acl_list>` / `--acl <acl_list>` / `-al <acl_list>` / `-a <acl_list>` (required): Comma-separated list of ACLs to add. 
  - Format: `type:name:permission[,type:name:permission...]`
  - Types: `user`, `group`
  - Permissions: `r` (READ), `w` (WRITE), `c` (CREATE), `d` (DELETE), `l` (LIST), `a` (ALL), `n` (NONE), `x` (READ_ACL), `y` (WRITE_ACL)
  - Example: `--acls user:user2:a,group:hadoop:r`

##### Example

```bash
ozone admin acl addacl volume myvolume --acls user:user1:rw,group:hadoop:r
ozone admin acl addacl bucket mybucket --acls user:user1:rw,group:hadoop:r
ozone admin acl addacl key o3fs://mybucket/mykey --acls user:user1:r
```

#### removeacl

Removes ACLs from an Ozone resource.

```bash
ozone admin acl removeacl <resource_type> <resource_name> --acls <acl_list>
```

##### Resource Types

- `volume`:  Applies to a volume.
- `bucket`: Applies to a bucket.
- `key`: Applies to a key.

##### Options

- `--acls <acl_list>` / `--acl <acl_list>` / `-al <acl_list>` / `-a <acl_list>` (required): Comma-separated list of ACLs to remove. 
  - Format: `type:name:permission[,type:name:permission...]`
  - Types: `user`, `group`
  - Permissions: `r` (READ), `w` (WRITE), `c` (CREATE), `d` (DELETE), `l` (LIST), `a` (ALL), `n` (NONE), `x` (READ_ACL), `y` (WRITE_ACL)
  - Example: `--acls user:user2:a,group:hadoop:r`

##### Example

```bash
ozone admin acl removeacl volume myvolume --acls user:user1:rw
ozone admin acl removeacl bucket mybucket --acls user:user1:rw
ozone admin acl removeacl key o3fs://mybucket/mykey --acls user:user1:r
```

#### setacl

Sets ACLs for an Ozone resource, replacing existing ACLs.

```bash
ozone admin acl setacl <resource_type> <resource_name> --acls <acl_list>
```

##### Resource Types

- `volume`:  Applies to a volume.
- `bucket`: Applies to a bucket.
- `key`: Applies to a key.

##### Options

- `--acls <acl_list>` / `--acl <acl_list>` / `-al <acl_list>` / `-a <acl_list>` (required): Comma-separated list of ACLs to set. 
  - Format: `type:name:permission[,type:name:permission...]`
  - Types: `user`, `group`
  - Permissions: `r` (READ), `w` (WRITE), `c` (CREATE), `d` (DELETE), `l` (LIST), `a` (ALL), `n` (NONE), `x` (READ_ACL), `y` (WRITE_ACL)
  - Example: `--acls user:user2:a,group:hadoop:r`

##### Example

```bash
ozone admin acl setacl volume myvolume --acls user:user1:r,group:hadoop:r
ozone admin acl setacl bucket mybucket --acls user:user1:r,group:hadoop:r
ozone admin acl setacl key o3fs://mybucket/mykey --acls user:user1:r