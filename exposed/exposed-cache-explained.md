# Exposed Cache Explained

## Short answer

Exposed uses caches because its DAO API is doing more than query generation.

It wants to provide:

- identity map semantics inside a transaction
- write-behind / unit-of-work behavior for entity changes
- reuse of loaded relations and eager-loaded collections
- cheaper repeated decoding, metadata lookups, and identifier processing

If you are using Exposed DAO, "the cache" usually means `EntityCache`.
That cache is transaction-scoped, not application-global.

## Why Exposed needs a cache

Without a cache, two reads of the same row inside one transaction could produce two different Kotlin objects:

```kotlin
transaction {
    val a = User[1]
    val b = User[1]

    check(a === b)
}
```

Exposed wants that `check(a === b)` to hold for DAO entities in the same transaction.
Otherwise you get weak identity, awkward dirty tracking, duplicate relation loads, and much easier stale-object bugs.

It also wants this style of code to work:

```kotlin
transaction {
    val user = User[1]
    user.name = "Alice"

    // No UPDATE yet.
    // The change is buffered in memory.

    User.find { Users.id eq 1 }.single()
    // Exposed flushes pending entity changes before running the query.
}
```

That means Exposed needs more than a read cache.
It needs a transaction-local identity map plus a write buffer.

## The main cache: `EntityCache`

`Transaction.entityCache` is created lazily per transaction:

```kotlin
val Transaction.entityCache: EntityCache by transactionScope { EntityCache(this) }
```

At a high level, `EntityCache` stores:

- `data[table][id] -> Entity`
- `inserts[table] -> pending new entities`
- `updates[table] -> dirty entities waiting to flush`
- `referrers[column][parentId] -> cached child collections`
- temporary initialization state for entities currently being constructed

So `EntityCache` is both:

- an identity map
- a unit-of-work buffer

## How reads work

### 1. Cache lookup first

`EntityClass.findById(id)` calls `testCache(id)` first.
If the entity is already present, Exposed returns that instance.

If not, it queries the database and then calls `wrap(id, row)`.

### 2. `wrap()` guarantees one entity instance per row per transaction

`EntityClass.wrap(id, row)` does:

- find the entity in `transaction.entityCache`
- if missing, create it
- assign `klass` and `db`
- store it in the cache

That is the core identity-map behavior.

### 3. `readValues` keeps the current loaded snapshot

Each `Entity` stores:

- `readValues`: the last materialized row state
- `writeValues`: unflushed changes

Reads prefer `writeValues` first, then fall back to `readValues`.
That lets Exposed show you in-memory updates before they are flushed.

## How writes work

Changing a DAO property does not immediately emit SQL.

When you set a column-backed property:

- Exposed writes the value into `Entity.writeValues`
- it schedules the entity in `EntityCache.updates`
- for new entities, it tracks them in `EntityCache.inserts`

Flush later turns buffered state into `INSERT` or `UPDATE`.

### Flush points

`EntityLifecycleInterceptor` is what makes this coherent.
Before many statements, it forces pending entity changes out to the database.

Important cases:

- before queries touching relevant DAO tables
- before commit
- before DDL
- before direct DSL `UPDATE`, `DELETE`, `UPSERT`, or `INSERT` that could invalidate DAO state

So the cache is part of Exposed's correctness model, not just a performance trick.

## How relation caching works

There are two related mechanisms.

### Transaction relation cache

`EntityCache.referrers` caches reverse relations and collections such as:

- one-to-many referrers
- eager-loaded child collections

This avoids repeating the same child query inside the transaction.

It also supports Exposed's eager-loading APIs such as `.load()` and `.with()`.

### Optional out-of-transaction reference cache

Each `Entity` also has a private `referenceCache`.
It is used only when `DatabaseConfig.keepLoadedReferencesOutOfTransaction = true`.

That mode allows already-loaded references to be accessed outside the transaction block.
Without that flag, loaded relations live only in the transaction cache.

## How Exposed avoids stale cached entities

A cache is only useful if it does not lie too much.
Exposed has explicit coherence logic for this.

### `wrapRow()` refreshes cached entities

If an entity instance already exists in the cache and Exposed reads a fresh `ResultRow`,
`EntityClass.wrapRow()` merges the new row data into the cached entity.

This matters for cases like `SELECT FOR UPDATE`.
Exposed had a regression around stale cached values here, and the fix was to always refresh the cached entity's row state with the fresh row data.

### Direct DSL writes invalidate DAO cache state

If you bypass DAO and run direct DSL `UPDATE`, `DELETE`, or `UPSERT`,
`EntityLifecycleInterceptor` clears or invalidates affected DAO cache entries and referrer caches.

That keeps DAO objects from pretending they still reflect the database after out-of-band writes.

### Explicit reload paths exist

When you really want to force a refresh, Exposed gives you tools such as:

- `reload()`
- `refresh()`
- `forceUpdateEntity()`

These are basically escape hatches around the identity map.

## Cache lifetime and eviction

For normal DAO entities, `EntityCache` lives inside the current transaction scope.

By default:

```kotlin
DatabaseConfig.maxEntitiesToStoreInCachePerEntity = Int.MAX_VALUE
```

So entity retention is unlimited unless you configure otherwise.

If you lower the limit, `EntityCache` uses a bounded `LinkedHashMap` per entity type and drops the oldest inserted entries when the limit is exceeded.

Important nuance:

- this is not a true LRU cache
- it behaves like bounded insertion-order eviction

Setting the limit to `0` effectively disables retention in `EntityCache.data`.

## Rollback behavior

Rollback is a place where ORM caches often go wrong.
Exposed explicitly cleans up mutable DAO state on rollback:

- referrer caches are cleared
- `writeValues` are cleared
- most cached `readValues` are cleared as well

That prevents stale mutable entity state from leaking into a new transaction attempt.

## Special case: `ImmutableCachedEntityClass`

This is the closest thing Exposed has to a cross-transaction entity cache.

`ImmutableCachedEntityClass` keeps `_cachedValues` per database and reuses them across transactions.

Why this is safe enough:

- the entity class is declared immutable
- Exposed can assume the loaded state is stable unless the cache is explicitly expired

How it works:

- `warmCache()` copies the per-database immutable cache into the current transaction cache
- on first warm-up, it materializes `all()` to populate that cache
- `expireCache()` clears the cached values
- creation of immutable cached entities also invalidates the global immutable cache

This is a different beast from normal `EntityCache`.
Normal DAO cache is transaction-local.
`ImmutableCachedEntityClass` intentionally stretches beyond one transaction.

## Cache comparison: Exposed vs JPA and Hibernate

The most important distinction is:

- a Hibernate `Session` is a larger runtime object
- the Hibernate first-level cache lives inside that session
- Exposed `EntityCache` is closer to the Hibernate first-level cache than to the whole Hibernate session

| Aspect | Hibernate `Session` | Hibernate first-level cache | Exposed DAO `EntityCache` |
| --- | --- | --- | --- |
| What it is | Main ORM runtime context for one unit of work | The session-internal identity/persistence cache | DAO cache bound to the current transaction |
| Scope | One `Session` | Same lifetime as the `Session` | Current transaction |
| Relationship to others | Contains the first-level cache and coordinates ORM behavior | Part of the `Session` | Standalone transaction-scoped DAO component |
| Main job | Manage entities, flushing, lazy loading, cascades, detachment, merge, action scheduling | Guarantee identity and hold managed state for dirty checking | Guarantee identity and buffer DAO inserts/updates |
| Cached things | Indirectly all managed ORM state through the session runtime | Managed entities, snapshots, managed collections | DAO entities, pending inserts, pending updates, referrer collections |
| Identity guarantee | Yes, via the session's persistence context | Yes | Yes |
| Write buffering | Yes | Yes | Yes |
| Lazy loading / proxies | Yes, central feature | Works with proxies/collections | No equivalent Hibernate-style proxy system |
| Detach / merge semantics | Yes | Participates in that model | Minimal, not a central model |
| Cascades / graph management | Yes | Participates indirectly | Much more limited |
| Cross-transaction reuse | No | No | No, except `ImmutableCachedEntityClass` |
| Query cache role | Not itself a query cache | Not a query cache | Not a query cache |
| Complexity level | High, full ORM runtime | Medium-high, part of ORM core | Lower, focused DAO support |

### Separate table for shared caches

Exposed also differs from Hibernate because Hibernate has optional shared cache layers that Exposed normally does not mirror.

| Aspect | Hibernate second-level cache | Hibernate query cache | Exposed |
| --- | --- | --- | --- |
| Scope | Across sessions | Across sessions | No general equivalent |
| Purpose | Reuse entity/collection state across sessions | Reuse query results across sessions | Mostly transaction-local cache only |
| Optional | Yes | Yes | Not applicable in the same sense |
| Identity owner | No, the session still owns identity | No | Not applicable |

### JPA persistence context

JPA itself is a specification, not a concrete cache implementation.
The JPA term is **persistence context**.

An `EntityManager` owns a persistence context.
Inside that context:

- one database row maps to one managed entity instance
- entity changes are tracked
- updates are usually flushed later, not immediately on every setter call
- entities can become detached when the context ends or when they are explicitly detached

So when people talk about the "JPA first-level cache", they usually mean the persistence context behavior provided by a JPA implementation.

### Hibernate first-level cache

Hibernate is the concrete ORM implementation.
Its `Session` is the equivalent of JPA's `EntityManager`, and the first-level cache lives inside that session.

Hibernate's first-level cache is richer than Exposed's DAO cache.
It typically manages:

- managed entity instances
- snapshots used for dirty checking
- managed associations and collections
- lazy proxies and collection wrappers
- action scheduling for inserts, updates, and deletes

Important properties:

- it is always session-scoped
- it guarantees identity inside the session
- it is central to Hibernate's ORM model
- it works together with cascades, flush modes, detachment, merge, lazy loading, and more

So Hibernate's first-level cache is not just a map of IDs to objects.
It is part of a full persistence-context runtime.

### Hibernate second-level cache

Hibernate's second-level cache is a separate, optional cache layer.
It is shared across sessions.

Its job is different from the first-level cache:

- it does not own live managed entity identity
- it stores reusable data across sessions
- it is usually backed by a cache provider

Typical cached content:

- entity state by primary key
- collections
- natural-id lookups

This cache is optional and explicitly configured.
It is not the same as the session cache.

### Hibernate query cache

Hibernate can also cache query results.
This is also optional.

Conceptually:

- query cache stores result sets, keys, IDs, or scalar outputs of queries
- entity materialization still depends on the first-level cache and often the second-level cache

So query cache is a different layer again.
It is not a substitute for the first-level cache.

### Exposed `EntityCache`

Exposed's DAO `EntityCache` is much narrower in scope.

It mainly provides:

- identity map behavior inside a transaction
- buffering of inserts and updates
- caching of referrer collections and eagerly loaded relations

What it does **not** try to be:

- a full JPA persistence context model
- a generic cross-transaction second-level entity cache
- a general query cache

This is why Exposed feels lighter.
The cache is there to make DAO semantics coherent, not to power a full ORM runtime with detached entities, merge semantics, lazy proxies, and provider-level cache layers.

### Direct interpretation

If you compare them strictly:

- JPA persistence context: specification concept
- Hibernate first-level cache: concrete, rich persistence-context implementation
- Hibernate second-level cache: optional shared cache across sessions
- Hibernate query cache: optional cache for query results
- Exposed `EntityCache`: transaction-local DAO cache with identity and write buffering

If you use only Exposed DSL, there is no DAO persistence context at all.
You only get smaller helper caches such as `ResultRow`, identifier, and metadata caches.

## Smaller caches outside DAO

If you read the Exposed source, you will find several other caches.
They are real, but they are not the main answer to your question.

### `ResultRow` cache

Each `ResultRow` has a tiny per-row cache for expression lookup and conversion.

Reason:

- avoid repeated `valueFromDB(...)`
- avoid repeated casting/decoding for the same expression

This is a micro-cache around row materialization.

### Identifier caches

`IdentifierManagerApi` caches things like:

- whether a token is a keyword
- whether an identifier needs quotes
- how it should be cased for the dialect

Reason:

- SQL rendering asks these questions constantly
- the answers are stable for repeated identifiers

### Metadata caches

Dialect metadata classes cache:

- table names
- schema names
- constraints
- indices

Reason:

- metadata access through JDBC or R2DBC is relatively expensive
- schema introspection should not repeat the same I/O unnecessarily

## Mental model

For a Kotlin expert, the cleanest model is:

- DSL layer: SQL builder with a few small optimization caches
- DAO layer: identity map + unit of work + relation cache
- immutable DAO layer: optional cross-transaction cache

So Exposed does not use a cache only to "go faster".
It uses caches because DAO semantics would be weaker and less correct without them.

## One-sentence answer

Exposed uses caches because its DAO API needs a transaction-local identity map and write-behind buffer, and then it adds smaller caches for row conversion, identifier handling, and metadata lookup to avoid repeating expensive work.

## Source trail

Start here if you want to read the implementation:

- `exposed-dao/src/main/kotlin/org/jetbrains/exposed/v1/dao/EntityCache.kt`
- `exposed-dao/src/main/kotlin/org/jetbrains/exposed/v1/dao/EntityClass.kt`
- `exposed-dao/src/main/kotlin/org/jetbrains/exposed/v1/dao/Entity.kt`
- `exposed-dao/src/main/kotlin/org/jetbrains/exposed/v1/dao/EntityLifecycleInterceptor.kt`
- `exposed-dao/src/main/kotlin/org/jetbrains/exposed/v1/dao/References.kt`
- `exposed-core/src/main/kotlin/org/jetbrains/exposed/v1/core/ResultRow.kt`
- `exposed-core/src/main/kotlin/org/jetbrains/exposed/v1/core/statements/api/IdentifierManagerApi.kt`
- `exposed-jdbc/src/main/kotlin/org/jetbrains/exposed/v1/jdbc/vendors/DatabaseDialectMetadata.kt`
- `exposed-tests/src/test/kotlin/org/jetbrains/exposed/v1/tests/shared/entities/EntityCacheTests.kt`
- `exposed-tests/src/test/kotlin/org/jetbrains/exposed/v1/tests/shared/entities/EntityCacheRefreshTests.kt`
- `exposed-tests/src/test/kotlin/org/jetbrains/exposed/v1/tests/shared/entities/ImmutableEntityTest.kt`
