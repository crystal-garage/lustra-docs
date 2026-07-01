# Select Clause

Start a low-level select query with `Lustra::SQL.select`.

```crystal
query = Lustra::SQL.select(:id, :email)
  .from(:users)
  .where(active: true)
  .order_by(id: :asc)
  .limit(20)

query.to_sql
# => SELECT "id", "email" FROM "users" WHERE "active" = TRUE ORDER BY "id" ASC LIMIT 20
```

When no selected columns are provided, Lustra renders `SELECT *`.

```crystal
Lustra::SQL.select.from(:users).to_sql
# => SELECT * FROM "users"
```

## Selecting Columns

Pass strings, symbols, raw SQL fragments, or aliases.

```crystal
Lustra::SQL.select(:id, :email).from(:users)

Lustra::SQL.select("COUNT(*) AS total").from(:users)

Lustra::SQL.select({uid: "user_id"}).from(:posts)
# => SELECT user_id AS uid FROM "posts"
```

Use `distinct` for `SELECT DISTINCT` or `DISTINCT ON`.

```crystal
Lustra::SQL.select(:role).distinct.from(:users)

Lustra::SQL.select(:id, :email).distinct(%("users"."id")).from(:users)
```

## Filtering

The low-level builder supports the same expression engine used by model collections.

```crystal
Lustra::SQL.select.from(:users).where { users.id > 100 }

Lustra::SQL.select.from(:users).where(email: "admin@example.com")

Lustra::SQL.select.from(:users).where("email LIKE ?", "%@example.com")

Lustra::SQL.select.from(:users).where("email LIKE :pattern", pattern: "%@example.com")
```

Plain string conditions are raw SQL.

```crystal
Lustra::SQL.select.from(:users).where("deleted_at IS NULL")
```

## Fetching Rows

`fetch` yields each row as `Hash(String, Lustra::SQL::Any)`.

```crystal
Lustra::SQL.select(:id, :email).from(:users).fetch do |row|
  puts row["email"]
end
```

Use `to_a` when you need all rows in memory.

```crystal
rows = Lustra::SQL.select(:id).from(:users).to_a
```

Use `fetch_first` or `fetch_first!` for one row.

```crystal
row = Lustra::SQL.select.from(:users).where(id: 1).fetch_first
```

Use `scalar` when the query returns one row and one column.

```crystal
count = Lustra::SQL.select("COUNT(*)").from(:users).scalar(Int64)
```

For large result sets, `fetch_with_cursor` streams rows through a PostgreSQL cursor.

```crystal
Lustra::SQL.select.from(:events).fetch_with_cursor(1_000) do |row|
  puts row["id"]
end
```

## Composing Write Queries

A select query with one table in `FROM` can be converted to an update or delete query. Lustra copies the table and `WHERE` clauses into the write query.

```crystal
inactive_users = Lustra::SQL.select.from(:users).where(active: false)

inactive_users.to_update.set(archived: true).execute
inactive_users.to_delete.execute
```
