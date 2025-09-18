# Chapter 7: Multi-Tenancy Client Pool

In our [previous chapter on the Hive Actor](06_hive_actor_.md), we met the specialist technician who communicates directly with the Hive Metastore (HMS). That technician used a simple pool of tools (connections), which worked fine because we assumed they were the only one working. But what happens on a busy day when multiple technicians, each working for a different client, need to use the central machinery at the same time?

This chapter introduces the **Multi-Tenancy Client Pool**, a sophisticated connection manager designed for a busy, secure, multi-user environment.

### The Problem: A Shared Workshop with Secure Tools

Imagine our service is a shared workshop, and the Hive Metastore is a powerful, dangerous machine in the center. Our workshop is run by a single master foreman (our service's proxy user, e.g., `catalog-service`).

Two different clients, Alice and Bob, send their projects to the workshop. The foreman needs to operate the machine on their behalf.
- When working on Alice's project, the machine must be configured with Alice's safety settings and permissions.
- When working on Bob's project, it must use Bob's settings.

The simple toolset from the last chapter is like a single wrench permanently configured for the foreman. It can't be reconfigured for Alice or Bob. We need a much smarter system: a secure tool cabinet where there's a specific drawer for each client. Alice's drawer contains tools pre-configured just for her, and Bob's drawer contains tools for him.

The `MultiTenancyClientPool` is this secure tool cabinet. It manages separate sets of connections (tools) for each user, ensuring that every action is performed with the correct user's identity and permissions.

### Meet the Key Concepts

Our "secure tool cabinet" operates on two main principles:

1.  **A Pool of Pools**: Instead of one large connection pool for everyone, the `MultiTenancyClientPool` manages many small, dedicated pools. There's one pool for Alice, one for Bob, and so on. When a request for Alice comes in, we go to her dedicated pool to get a connection.

2.  **Delegation Tokens (The Permission Slip)**: In a secure environment (using Kerberos), the foreman can't just tell the machine, "I'm working for Alice now." The machine will demand proof. The foreman must first go to the security office (the HMS itself) and get a signed, temporary permission slip called a **delegation token**. This token says, "I, the trusted foreman, am authorized to act on behalf of Alice for the next hour." When the foreman uses a tool from Alice's drawer, they attach this token, and the machine accepts the operation.

### A Request's Journey Through the Pool

Let's follow a request from Alice to see how this sophisticated pool works.

1.  A request for Alice arrives. The code needs a connection to the HMS to proceed.
2.  It asks the `MultiTenancyClientPool` for a connection.
3.  The pool checks its cache of user-specific pools. "Do I have a connection pool for Alice?"
4.  **Cache Miss!** This is the first time we've seen Alice. The pool now performs a critical setup:
    a. It uses the powerful foreman's identity to connect to the HMS security office.
    b. It requests a delegation token for Alice.
    c. The HMS returns a unique, temporary token.
    d. The pool creates a *brand new, small connection pool just for Alice*, configured to use this token for all future authentication.
    e. It stores this new pool in its cache, keyed by Alice's identity.
5.  The pool borrows a fresh connection from Alice's newly created pool.
6.  The request is sent to the HMS over this connection. The HMS inspects the token, recognizes it's for Alice, and applies her permissions.
7.  When the operation is complete, the connection is returned to *Alice's pool*, ready for her next request.

When Bob's request arrives next, the same process happens, but this time a separate pool is created just for him.

### The Big Picture: Creating a User-Specific Connection

This diagram shows the one-time setup that happens the first time a user like Alice makes a request.

```mermaid
sequenceDiagram
    participant Actor as Hive Actor
    participant MTPool as MultiTenancyClientPool
    participant ServiceUser as Service 'Foreman' UGI
    participant HMS as Hive Metastore

    Note over Actor: Request for user 'alice' arrives
    Actor->>MTPool: Get me a connection
    MTPool->>MTPool: Which user? 'alice'. Check cache.
    Note over MTPool: Cache miss! Must create new pool for 'alice'.
    MTPool->>ServiceUser: Act as the powerful service user
    ServiceUser->>HMS: Please give me a delegation token for 'alice'
    HMS-->>ServiceUser: Here is the token
    ServiceUser-->>MTPool: Returns 'alice''s token
    MTPool->>MTPool: Create a new HiveClientPool for 'alice' configured with this token
    MTPool->>HMS: Borrow connection from 'alice''s pool & execute action
    HMS-->>MTPool: Returns result
    MTPool-->>Actor: Returns result
```
Every subsequent request from Alice will be a "cache hit," immediately finding and using her dedicated pool without needing a new token.

### Under the Hood: The Code Path

The magic starts with a special version of the Hive Actor that knows to use our multi-tenant pool.

#### Step 1: Choosing the Right Pool

The `HiveCatalogActorMT` (Multi-Tenant) simply overrides one method to ensure it uses our smart pool instead of a basic one.

```java
// File: iceberg/hive/HiveCatalogActorMT.java
@Override
protected ClientPool<IMetaStoreClient, TException> createPool(Map<String, String> properties) {
  // Instead of a default pool, instantiate our special multi-tenant one.
  return new MultiTenancyClientPool(getConf(), properties);
}
```
This is the simple switch that puts our "secure tool cabinet" in place.

#### Step 2: Finding the User's Pool

When a connection is needed, the `MultiTenancyClientPool`'s `clientPool()` method is called. This is where the "pool of pools" logic lives.

```java
// File: iceberg/hive/MultiTenancyClientPool.java
@Override
HiveClientPool clientPool() {
  return CachedClientPool.clientPoolCache().get(
      // 1. Get a unique key for the current user (e.g., "alice")
      extractKey(..., conf),
      // 2. If no pool exists, create a token-based one
      k -> new TokenAuthHiveClientPool(clientPoolSize, conf)
  );
}
```
This code does two things:
1.  `extractKey(...)` figures out who the current user is. This is our key to the cache.
2.  `clientPoolCache().get(...)` is the "find-or-create" step. It looks for a pool matching the user's key. If it can't find one, it runs the code to create a new `TokenAuthHiveClientPool`.

#### Step 3: Getting the Delegation Token

The `TokenAuthHiveClientPool` is a special pool that knows how to get permission slips. This logic is in its `setDelegationToken` method.

```java
// File: iceberg/hive/MultiTenancyClientPool.java (inner class)
private void setDelegationToken() {
  try {
    // 1. Get current user's identity (e.g., UGI for 'alice')
    UserGroupInformation currentUgi = SecurityUtils.getUGI();

    // 2. Act as the powerful service user (the 'foreman')
    IMetaStoreClient client = currentUgi.getRealUser().doAs(
        () -> new HiveMetaStoreClient(hiveConf));

    // 3. Ask for a token for the proxy user ('alice')
    String token = client.getDelegationToken(currentUgi.getUserName(), ...);

    // 4. Store the token so it can be used for new connections
    SecurityUtils.setTokenStr(currentUgi, token, ...);
  } catch (Exception e) { /* ... error handling ... */ }
}
```
This is the core of the security handshake, translated into code. It temporarily uses the service's power to get a permission slip for the end-user.

#### Step 4: Creating a Secure Connection

Finally, whenever this pool needs to create a brand new connection, it first ensures the token is ready.

```java
// File: iceberg/hive/MultiTenancyClientPool.java (inner class)
@Override
protected IMetaStoreClient newClient() {
  // First, get the permission slip for this user.
  setDelegationToken();
  // Now, create the actual connection. It will automatically
  // present the token to the HMS for authentication.
  return super.newClient();
}
```
This guarantees that any connection created within Alice's pool is securely identified as belonging to Alice, and the Hive Metastore will enforce her specific permissions.

### Conclusion

In this chapter, we've explored the `Multi-Tenancy Client Pool`, a critical component for building a secure, robust, multi-user data catalog.

The key takeaways are:
-   It provides a **"pool of pools"**, maintaining a separate, dedicated connection pool for each user.
-   It handles the complex security dance of acquiring **delegation tokens**, allowing a trusted service to securely act on behalf of end-users.
-   This enables true multi-tenancy, where security policies are enforced by the Hive Metastore based on the actual user making the request, not a generic service account.

This concludes our journey through the core components of the Iceberg HMS REST Catalog. From handling web requests with the [HMS Catalog Server & Servlet](01_hms_catalog_server___servlet_.md) to managing cached metadata with the [Hive Caching Catalog](04_hive_caching_catalog_.md) and finally to handling secure, multi-tenant connections, you now have a foundational understanding of how this powerful system works from end to end.

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)