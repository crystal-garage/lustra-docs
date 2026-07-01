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
average_time = User.query.avg("time_connected", Float64)
total_score = User.query.sum("score")
```

`min`, `max`, and `avg` require the expected return type. `sum` returns `Float64`.

## Custom Aggregates

Use `agg` for custom aggregate expressions:

```crystal
median_age = User.query.agg("PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY age)", Float64)
```

`agg` returns one scalar value. For grouped aggregate rows, use `select`, `group_by`, and `fetch`/`pluck` instead.

## Exists

`exists?` checks whether at least one row exists:

```crystal
if User.query.where(active: true).exists?
  puts "There are active users"
end
```

`exists?` always queries the database. It is useful when you want a direct existence check instead of loading records.
