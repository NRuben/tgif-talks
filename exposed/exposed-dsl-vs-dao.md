# Exposed DSL vs DAO

## Short answer

For most projects, `DSL` is the better default.

`DAO` is a good fit when:

- you are using `JDBC`
- your domain model is simple
- most operations are straightforward CRUD
- you want an object-oriented style

## Decision table

| Question | DSL | DAO | Better choice |
| --- | --- | --- | --- |
| Default starting point | SQL-first, explicit, flexible | Entity-first, more convenient for CRUD | DSL |
| Complex joins, aggregates, reporting queries | Strong fit | Usually more awkward | DSL |
| Bulk updates / deletes / inserts | Strong fit | Less natural | DSL |
| Predictable SQL behavior | Easier to reason about | More lifecycle/cache behavior involved | DSL |
| Simple CRUD by ID | More verbose | Very convenient | DAO |
| Navigating object relations | Manual or semi-manual | More natural | DAO |
| Learning curve for SQL-minded developers | Usually lower | Can feel nicer only if you want entity-style access | DSL |
| JDBC support | Yes | Yes | Depends |
| R2DBC support | Yes | No | DSL |
| Fit for larger or more query-heavy applications | Better | Usually worse | DSL |
| Fit for small internal tools or simple domain apps | Good | Often very pleasant | DAO |

## Short rule

- Use `DSL` if you care about control, explicitness, complex queries, bulk operations, or `R2DBC`.
- Use `DAO` if you are on `JDBC` and mostly want convenient entity-style CRUD.

## Hard limitation

`DAO` is only compatible with `exposed-jdbc` and does not work with `exposed-r2dbc`.

## Practical recommendation

If you are unsure, start with `DSL`.

It scales better once queries become non-trivial, stays closer to SQL, and is easier to reason about in terms of generated statements and performance.

Choose `DAO` only when the extra convenience is actually useful and you accept the entity lifecycle and cache behavior that comes with it.
