# Delete Clause

Start a delete query with `Lustra::SQL.delete`.

```crystal
Lustra::SQL.delete(:sessions)
  .where { expires_at < Time.utc(2026, 1, 1) }
  .execute
```

Use `execute_and_count` when you need the number of deleted rows.

```crystal
deleted = Lustra::SQL.delete(:sessions)
  .where { expires_at < Time.utc(2026, 1, 1) }
  .execute_and_count
```

## Filtering

Delete queries include the same `where` helpers as select queries.

```crystal
Lustra::SQL.delete(:users).where(id: 10)

Lustra::SQL.delete(:users).where("email LIKE ?", "%@example.com")

Lustra::SQL.delete(:users).where("email LIKE :pattern", pattern: "%@example.com")

Lustra::SQL.delete(:users).where { users.active == false }
```

Plain string conditions are raw SQL. Use them only for trusted SQL.

```crystal
Lustra::SQL.delete(:sessions).where("expires_at < NOW()")
```

## From a Collection or Select Query

Model collections and select builders can be converted to delete queries with `to_delete`. The new query reuses the source table and `WHERE` clauses.

```crystal
User.query
  .where(active: false)
  .to_delete
  .execute_and_count
```

`to_delete` is available only when the source query has exactly one table in `FROM` and does not use a subquery as its table.

## Returning Values

Use `execute_returning` to get deleted columns as typed tuples:

```crystal
rows = Lustra::SQL.delete(:users)
  .where(id: 10)
  .execute_returning({id: Int64, email: String})
# => Array(Tuple(Int64, String))
```

Types must match the PostgreSQL result columns. Tuple values follow the declared column order; returned row order is not guaranteed. No matching rows produces an empty array.

For raw hashes, use `.returning("id, email").fetch` with a block instead.
