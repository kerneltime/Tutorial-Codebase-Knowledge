# Chapter 3: Authorization Interceptor

In the [previous chapter](02_gravitino_server_application_.md), we saw how the `GravitinoServer` acts like a library director, starting up the server and registering all our "librarian" API endpoints. Now that the library is open, we need to think about security. Not everyone should be allowed to go into the rare books section or scribble in the archives!

This is where the Authorization Interceptor comes in. It's the core security mechanism that enforces access control. Think of it as a vigilant security guard posted at the entrance of every important room in our library. Before letting anyone in, the guard checks their ID and makes sure their name is on the approved list for that specific room.

### The Security Guard Analogy

Imagine a highly secure building with many restricted areas.

*   **API Endpoints (`...Operations` methods):** These are the restricted rooms (e.g., `deleteTable`, `createCatalog`).
*   **The User:** This is the person trying to enter a room.
*   **The Authorization Interceptor:** This is the security guard standing at the door of every room.
*   **`@AuthorizationExpression` Annotation:** This is the "Access Rules" plaque posted on the door, telling the guard who is allowed inside (e.g., "Only users in the 'Admins' group can enter").

The guard doesn't need to know what happens inside the room. Their only job is to check the rules on the door and the person's credentials, and then either grant or deny access. This is a powerful way to enforce security consistently across the entire application.

### Our Goal: Protecting an Endpoint

Let's set a simple goal: we want to understand how the server prevents an unauthorized user from deleting a table. We will see how the Authorization Interceptor steps in to protect the `deleteTable` method before it can do any damage.

### How It Works: The Interception Process

When a user tries to call a protected method, the interceptor "catches" the call before the method's code can run.

Let's trace a request to delete a table:

1.  **Request Arrives:** A user sends a `DELETE` request to the server to delete a table.
2.  **Routing:** The server routes this request to the `deleteTable` method inside `TableOperations.java`.
3.  **INTERCEPTION!** Before a single line of the `deleteTable` method is executed, the **Authorization Interceptor** steps in.
4.  **Rule Check:** The interceptor looks at the `deleteTable` method and finds a security annotation on it, something like `@AuthorizationExpression("hasRole('ADMIN')")`.
5.  **Decision Time:**
    *   **Access Granted:** If the current user has the 'ADMIN' role, the interceptor steps aside and lets the `deleteTable` method run as normal.
    *   **Access Denied:** If the user does not have the 'ADMIN' role, the interceptor blocks the request completely. It immediately sends back a "Forbidden" error message to the user. The `deleteTable` method is never called.

This entire process is automatic. As developers, we just need to put the right security "plaque" (annotation) on our methods.

Here's a diagram of that flow:

```mermaid
sequenceDiagram
    participant User
    participant Server as Gravitino Server
    participant Interceptor as Auth Interceptor
    participant TO as TableOperations

    User->>+Server: DELETE /.../tables/my_table
    Server->>+Interceptor: Call deleteTable(...)
    Note right of Interceptor: Intercepts the call
    Interceptor->>Interceptor: Check @AuthorizationExpression rule
    alt User has permission
        Interceptor->>+TO: proceed() to deleteTable(...)
        TO-->>-Interceptor: Deletion successful
        Interceptor-->>-Server: Success
    else User does NOT have permission
        Interceptor-->>-Server: Return 403 Forbidden Error
        Note left of TO: deleteTable() is never called!
    end
    Server-->>-User: Returns final response
```

### A Look at the Code: Declaring the Security Rule

Let's go back to the `createSchema` method we saw in the first chapter. You may have noticed this annotation:

```java
// File: src/main/java/org/apache/gravitino/server/web/rest/SchemaOperations.java

@POST
@AuthorizationExpression(
    expression = "hasPermission(p_metalake, 'CREATE_SCHEMA', p_catalog)"
)
public Response createSchema(...) {
    // ... method logic here ...
}
```

This is the "Access Rules" plaque in action. The `@AuthorizationExpression` annotation tells the interceptor: "Before running this method, check if the current user has the 'CREATE_SCHEMA' permission on the catalog they are trying to modify."

The interceptor will parse this `expression` and perform the check automatically.

### Under the Hood: The Interceptor's Logic

How does this magic happen? It's managed by a class called `GravitinoInterceptionService`. This service is registered when the server starts up, as we saw in the [Gravitino Server Application](02_gravitino_server_application_.md) chapter.

#### Step 1: Telling the Server Which Classes to Watch

First, the service tells the server which classes need a "security guard." It provides a list of all the `...Operations` classes.

```java
// File: src/main/java/org/apache/gravitino/server/web/filter/GravitinoInterceptionService.java

public class GravitinoInterceptionService implements InterceptionService {
    @Override
    public Filter getDescriptorFilter() {
        // This is a list of classes to apply interception to.
        return new ClassListFilter(
            ImmutableSet.of(
                MetalakeOperations.class.getName(),
                CatalogOperations.class.getName(),
                SchemaOperations.class.getName(),
                // ... and all other ...Operations classes
            )
        );
    }
    // ...
}
```

This code effectively says, "Assign a security guard to every door in the Metalake, Catalog, and Schema departments."

#### Step 2: The Guard's Instructions (`invoke` method)

The actual guard is a `MethodInterceptor`. Its `invoke` method contains the core logic that runs *before* every single API method in the watched classes.

Let's look at a very simplified version of what this `invoke` method does.

```java
// File: src/main/java/org/apache/gravitino/server/web/filter/GravitinoInterceptionService.java

private static class MetadataAuthorizationMethodInterceptor implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation methodInvocation) throws Throwable {
        // 1. Get the method that was called (e.g., createSchema)
        Method method = methodInvocation.getMethod();

        // 2. Find the security annotation on it
        AuthorizationExpression annotation = method.getAnnotation(AuthorizationExpression.class);

        // 3. If there is an annotation, perform the security check
        if (annotation != null) {
            boolean isAllowed = checkPermissions(annotation.expression(), ...);

            // 4. If the check fails, block the request
            if (!isAllowed) {
                return buildNoAuthResponse("Access Denied!"); // Stop here!
            }
        }

        // 5. If everything is OK, let the original method run
        return methodInvocation.proceed();
    }
}
```

Breaking it down:

1.  **Get Method:** It identifies which "room" the user is trying to enter.
2.  **Find Annotation:** It reads the "Access Rules" plaque on the door.
3.  **Check Permissions:** It evaluates the rules against the user's credentials. This is the heart of the security check.
4.  **Block if Needed:** If the user is not authorized, it builds and returns an error response immediately.
5.  **Proceed:** If the user is authorized, `methodInvocation.proceed()` is called. This special command means "Okay, you're clear. Go ahead and run the actual method now."

### Conclusion

In this chapter, we learned about the Authorization Interceptor, Gravitino's primary security mechanism. It acts like a security guard that automatically protects our API endpoints.

*   It uses a technique called **interception** to check requests *before* the target method is executed.
*   Security rules are declared cleanly using the **`@AuthorizationExpression`** annotation directly on the API methods.
*   This system ensures that security is enforced consistently and that protected code is never run by an unauthorized user.

Now we understand how the server handles both valid requests and unauthorized requests. But what about other types of errors, like a user trying to create a schema that already exists, or providing invalid input? How does the server report these problems in a clean and uniform way?

We'll explore this in the next chapter on error handling.

Next: [Centralized Exception Handling](04_centralized_exception_handling_.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)