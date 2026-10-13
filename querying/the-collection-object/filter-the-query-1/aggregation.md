# Aggregation

Collections expose common SQL aggregate helpers.

## Count

`count` returns the number of matching rows:

```crystal
user_count = User.query.count
active_count = User.query.where(active: true).count
```

You can choose the return type:

```crystal
count = User.query.count(Int32)
```

When the query contains `limit`, `offset`, or `group_by`, Lustra wraps the query in a subquery so the count applies to the filtered result set.

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

`exists?` always queries the database. It is useful when you want a direct existence check instead of loading records.

`count` and `exists?` have different costs: `count` asks for row cardinality, while `exists?` only checks that at least one row matches.
