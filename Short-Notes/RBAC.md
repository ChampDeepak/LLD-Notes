# RBAC — Role-Based Access Control

## The Problem

In v0, the proxy checked roles directly:

```java
if (user.getRole() != UserRole.ADMIN) {
    throw new ValidationException("Admin role required");
}
```

This violates the **Open/Closed Principle**. If a new role `OPERATOR` arrives that
should also add shows, you must modify every proxy's if-else block:

```java
if (user.getRole() != UserRole.ADMIN && user.getRole() != UserRole.OPERATOR) { ... }
```

Every new role = changes in every proxy class. Not scalable.

## The Solution

**Check permissions, not roles.** Instead of asking "who are you?", ask "can you do this?"

### Step 1 — Define a Permission enum

Each permission maps to a single action in the system:

```java
public enum Permission {
    ADD_SHOW, ADD_MOVIE, ADD_THEATER
}
```

### Step 2 — Roles own their permissions

Each role declares what it can do via `EnumSet`:

```java
public enum UserRole {
    CUSTOMER(EnumSet.noneOf(Permission.class)),
    OPERATOR(EnumSet.of(Permission.ADD_SHOW)),
    ADMIN(EnumSet.allOf(Permission.class));

    private final Set<Permission> permissions;

    UserRole(Set<Permission> permissions) {
        this.permissions = permissions;
    }

    public boolean hasPermission(Permission p) {
        return permissions.contains(p);
    }
}
```

### Step 3 — Proxy checks permission

```java
private User authorize(String userEmail, Permission permission) {
    User user = db.getUserByEmail(userEmail);
    if (user == null) throw new ValidationException("User not found");
    if (!user.getRole().hasPermission(permission)) {
        throw new ValidationException(permission + " permission required");
    }
    return user;
}

public Show addShowAsUser(String userEmail, ...) {
    authorize(userEmail, Permission.ADD_SHOW);
    return service.addShow(...);
}
```

## Why this works

| Event                  | Old design (role check) | New design (permission check) |
|------------------------|-------------------------|-------------------------------|
| Add OPERATOR role      | Edit every proxy's if-else | Add one enum entry with its permission set |
| Add new permission     | Add new if-else blocks | Add enum constant, assign to roles |
| Remove a role          | Hunt down all checks | Remove enum entry |

The proxy classes **never change** when roles are added/removed. Only the `UserRole` enum
and `Permission` enum evolve — the rest of the system is **closed for modification, open
for extension**.

## Key Insight

The indirection through permissions decouples two things that change independently:
- **Who exists** in the system (roles) — changes when org structure evolves
- **What is allowed** (permissions) — changes when features are added

Without this indirection, every role change ripples through authorization logic.
With it, each concern has exactly one place to change.
