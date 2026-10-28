# Aggregation

Collections expose common SQL aggregate helpers.

## Count

`count` returns the number of rows produced by the collection's `SELECT` query:

```crystal
user_count = User.query.count
active_count = User.query.where(active: true).count
```

You can choose the return type:

```crystal
count = User.query.count(Int32)
```

Lustra counts a wrapped copy of the full query. The selection, `distinct`,
`group_by`, `having`, `limit`, and `offset` all affect the result being counted.
For example, a grouped query counts groups, and a distinct selection counts
unique selected rows:

```crystal
user_groups = Post.query.select(:user_id).group_by(:user_id).count
# => 3_i64, if posts form 3 user_id groups

unique_emails = User.query.select(:email).distinct.count
# => 20_i64, if users have 20 distinct email values

page_size = User.query.order_by(:id).limit(10).offset(20).count
# => 5_i64, if there are 25 users (20 skipped, 5 remaining)
```

`page_size` is at most 10; it counts only the rows remaining after the offset
and limit.

An aggregate selection without grouping or `HAVING` produces one result row,
even when no source rows match. Counting that query therefore returns 1:

```crystal
result_rows = User.query.select("COUNT(*) AS total").count # => 1
```

Use `User.query.count` directly when you want the number of users.

Ordering is preserved when it affects which rows are selected, including
limits, offsets, and `DISTINCT ON`. Ordinary unpaginated counts omit unnecessary
ordering.

## Min, Max, Avg, and Sum

```crystal
max_id = User.query.max("id", Int64)
min_id = User.query.min("id", Int64)
average_id = User.query.avg("id", PG::Numeric).to_f
total_ids = User.query.sum("id")
```

`min`, `max`, and `avg` require the expected return type. `sum(field)` returns `Float64`; `sum(field, Type)` returns the requested type.

The return type must match the database result. For integer columns, `avg` returns `PG::Numeric`; call `to_f` on the result when you need a `Float64`.

`sum` does not sanitize the `field` input, so pass trusted SQL fragments only.

```crystal
total_post_ids = Post.query.where(published: true).sum("id")
# => Float64
```

Use the typed `sum` overload when the result should stay in a specific numeric type:

```crystal
total_post_ids = Post.query.where(published: true).sum("id", Int64)
# => Int64
```

The type must match PostgreSQL's aggregate result type. The typed overload returns zero when there are no matching values.

## Source Expressions and Selected Aliases

`sum`, `min`, `max`, and `avg` wrap queries containing a selection, `distinct`, `group_by`, `limit`, or `offset` in a subquery. Ordering is preserved so the aggregate uses the intended rows.

Pass `Lustra::SQL::Column` when the expression refers to a source table:

```crystal
query = User.query.order_by(:id).limit(10)
total_ids = query.sum(Lustra::SQL::Column.new("users.id"))
```

Lustra evaluates the expression inside the source query before aggregating the result. The expression must be valid in that query, including its grouping constraints. String expressions remain trusted SQL fragments; `SQL::Column` does not sanitize them.

Pass a string when aggregating a selected result column or alias:

```crystal
query = User.query.select("id * 2 AS doubled_id").order_by(:id).limit(10)
total = query.sum("doubled_id")
```

In a wrapped query, a qualified string such as `"users.id"` is evaluated outside the source query, where the original table is no longer in scope. Use `SQL::Column` for that reference.

With plain `DISTINCT`, source expressions passed as `SQL::Column` raise `Lustra::SQL::QueryBuildingError`, because adding a projection could change which rows are distinct. Aggregate a selected column or alias as a string instead:

```crystal
total = User.query.select("id").distinct.sum("id")
```

This restriction does not apply to `DISTINCT ON`.

## Custom Aggregates

Use `agg` for custom aggregate expressions:

```crystal
median_age = User.query.agg("PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY age)", Float64)
max_updated_at = User.query.agg("MAX(updated_at)", Time)
```

`agg` returns one scalar value. `agg` should not be used with `group_by` unless your intent is to wrap grouped rows first and aggregate that result as a whole.
For grouped results, use `select`, `group_by`, and `fetch`/`pluck`.

`agg` also handles paginated/limited queries by wrapping them in a subquery.

## Exists

`exists?` checks whether at least one row exists:

```crystal
if User.query.where(active: true).exists?
  puts "There are active users"
end
```

`exists?` checks the database without loading models, except when the query has `LIMIT 0`: it returns `false` without executing SQL.

`count` and `exists?` have different costs: `count` asks for row cardinality, while `exists?` only checks that at least one row matches.
