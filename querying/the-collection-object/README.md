# The Collection Object

Each model has a generated collection class, such as `User::Collection`.

Create a collection with `query`:

```crystal
users = User.query
```

A collection represents a PostgreSQL `SELECT` query:

```sql
SELECT * FROM users
```

The query is not sent to PostgreSQL when the collection is created. It is executed when you use a terminal helper such as `each`, `to_a`, `first`, `count`, `any?`, or `empty?`.

```crystal
users = User.query.where(active: true)

users.each do |user|
  puts user.email
end
```

## Refining Queries

Collections include Lustra's select-builder behavior, so you can refine the SQL:

```crystal
users = User.query
  .where(active: true)
  .order_by(created_at: "DESC")
  .limit(20)
```

Common refinement methods include `select`, `where`, `join`, `group_by`, `having`, `order_by`, `limit`, `offset`, `distinct`, and CTE/window helpers.

## Mutability

Collections are mutable. Query-refinement methods change the collection they are called on and usually return the same collection for chaining.

```crystal
users = User.query
users.select("id")
users.select("email")

puts users.to_sql
# SELECT id, email FROM "users"
```

This matters when reusing a collection variable:

```crystal
users = User.query.where(active: true)
admins = users.where(role: "admin")

# users and admins now point to the same refined collection.
```

Use `dup` when you want to keep the original query:

```crystal
users = User.query.where(active: true)
admins = users.dup.where(role: "admin")

users.to_sql  # active users
admins.to_sql # active admins
```

Create duplicated branches before attaching generated `with_*` eager-loading
helpers. Eager-loading work belongs to the collection on which `with_*` was
called, so duplicating that collection afterward is not a safe way to branch it.

## Terminal Helpers

Terminal helpers execute the query or derive a result from it:

| Helper | Result |
| :--- | :--- |
| `each` | Iterates over model records. |
| `to_a` | Loads all matching records into an array. |
| `first` / `first!` | Loads one record. |
| `last` / `last!` | Loads one record from the end of the ordered query. |
| `[]` / `[]?` | Loads a record at an offset. |
| `[](range)` | Loads a range into an array. |
| `any?` / `empty?` | Checks whether at least one row exists. |
| `count` | Runs a count query unless the collection already has a cached result. |

Terminal helpers are the boundary where SQL is sent to PostgreSQL. Helpers that
temporarily add a projection, filter, ordering, limit, or offset restore the
collection query before returning. A collection can therefore be reused after
`any?`, `empty?`, `pluck`, `first`, `last`, `find`, `find_by`, or array-style
access without retaining those temporary changes.

## Cached Results

Some association eager-loading paths attach cached records to a collection. When a cached result is present, terminal helpers such as `each`, `any?`, `empty?`, and `count` can use that cached result instead of issuing a new SQL query.

Refining a collection clears cached results because the SQL result set has changed.
