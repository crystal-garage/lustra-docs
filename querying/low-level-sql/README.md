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

## Query Plans

Use `explain` to inspect PostgreSQL's plan for a query.

```crystal
plan = User.query
  .where(active: true)
  .order_by(created_at: :desc)
  .explain

puts plan
```

Use `explain_analyze` when you need actual execution timing and row counts:

```crystal
plan = User.query
  .where(active: true)
  .order_by(created_at: :desc)
  .explain_analyze

puts plan
```

`explain_analyze` runs the query. Be careful with write queries because
PostgreSQL executes them to collect the plan statistics. For write-query
investigation, use a transaction and roll it back.

## Reporting Queries

Use `Lustra::SQL` when a query returns report data instead of model records.
This is useful for dashboards, charts, CTEs, and derived tables.

```crystal
counts = {} of String => Int64

Repository.query
  .select(
    "date_trunc('month', created_at)::date AS year_month",
    "COUNT(*) AS count"
  )
  .group_by("year_month")
  .order_by("year_month", :asc)
  .fetch do |row|
    counts[row["year_month"].as(Time).to_s("%Y-%m")] = row["count"].as(Int64)
  end
```

For a CTE, build one select query and pass it to `with_cte`:

```crystal
month_series = Lustra::SQL.select(<<-SQL)
  generate_series(
    '2012-11-01'::date,
    (SELECT date_trunc('month', MAX(created_at)) FROM repositories),
    '1 month'::interval
  )::date AS year_month
  SQL

Lustra::SQL
  .select({
    year_month:       "ms.year_month",
    cumulative_count: "(SELECT COUNT(*) FROM repositories WHERE date_trunc('month', created_at)::date <= ms.year_month)",
  })
  .with_cte({month_series: month_series})
  .from("month_series ms")
  .order_by("ms.year_month")
  .fetch do |row|
    puts "#{row["year_month"]}: #{row["cumulative_count"]}"
  end
```

For grouped data from a derived table, pass the derived table SQL to `from`:

```crystal
Lustra::SQL
  .select(
    "dependency_count",
    "COUNT(*) AS repository_count"
  )
  .from(<<-SQL)
    (
      SELECT
        r.id,
        COUNT(rel.id) AS dependency_count
      FROM repositories r
      LEFT JOIN relationships rel ON r.id = rel.master_id
      GROUP BY r.id
    ) AS repo_dependency_count
    SQL
  .group_by("dependency_count")
  .order_by("dependency_count", :asc)
  .fetch do |row|
    puts "#{row["dependency_count"]}: #{row["repository_count"]}"
  end
```

Prefer model collections when the result is a set of model records. Prefer
`Lustra::SQL` when the result is a custom row shape.

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

Raw SQL fragments are useful for internal expressions such as aggregate
ordering, PostgreSQL functions, and maintenance queries:

```crystal
Repository.query.order_by(
  "(select COUNT(*) from relationships r WHERE r.dependency_id = repositories.id)",
  :desc
)
```

Do not interpolate user input into raw SQL strings. Use expression helpers or
parameterized raw fragments instead:

```crystal
email = params["email"]

User.query.where { users.email == email }
Lustra::SQL.select.from(:users).where(Lustra::SQL.raw("email = ?", email))
```
