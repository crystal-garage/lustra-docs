# Writing low-level SQL

Lustra's model collections are built on top of a lower-level SQL builder. You can use it directly when you need a query that does not naturally belong to a model, when you want to compose SQL fragments, or when you need to run `INSERT`, `UPDATE`, or `DELETE` statements without instantiating model objects.

```crystal
Lustra::SQL.select.from(:users).where(active: true).to_sql
# => SELECT * FROM "users" WHERE "active" = TRUE
```

The low-level builders are mutable. Query-refinement methods such as `select`, `from`, `where`, `set`, `values`, `limit`, and `order_by` update the current builder and return it for chaining. Use `dup` before refining a `SelectBuilder` that you also need to reuse.

## Running Queries

For `SELECT` queries, use `fetch`, `to_a`, `fetch_first`, `scalar`, `pluck`, or `pluck_col` depending on the result shape you need.

```crystal
Lustra::SQL.select("id", "email").from(:users).fetch do |row|
  puts row["email"]
end
```

For write queries, use `execute` or `execute_and_count`.

```crystal
affected = Lustra::SQL.update(:users)
  .set(active: false)
  .where { last_seen_at < Time.utc(2024, 1, 1) }
  .execute_and_count
```

`Lustra::SQL.execute(sql)` can run a raw SQL string directly. Prefer the query builders or parameterized raw fragments when user-provided values are involved.

## Raw Fragments

`Lustra::SQL.raw` creates an SQL fragment with escaped values. It supports positional placeholders and named placeholders.

```crystal
Lustra::SQL.raw("email = ?", "admin@example.com")
# => email = 'admin@example.com'

Lustra::SQL.raw("id = :id", id: 10)
# => id = 10
```

Use `Lustra::SQL.escape` for identifiers such as table or column names, and `Lustra::SQL.sanitize` for literal values.

```crystal
Lustra::SQL.escape(:order)
# => "order"

Lustra::SQL.sanitize("l'hotel")
# => 'l''hotel'
```

Plain string conditions are inserted as SQL. Use them only for trusted SQL.

```crystal
Lustra::SQL.select.from(:users).where("deleted_at IS NULL")
```
