# Window Functions and CTEs

Lustra collections and low-level select queries can be used as subqueries and common table expressions.

## Common Table Expressions

PostgreSQL Common Table Expressions \(CTEs\) define named temporary result sets for a larger query.

Use `with_cte(name, query)`:

```crystal
active_users = User.query
  .select("id", "email")
  .where(active: true)

Lustra::SQL.select
  .with_cte("active_users", active_users)
  .from(:active_users)
  .fetch do |row|
    puts row["email"]
  end
```

You can also pass a `NamedTuple`:

```crystal
recent_posts = Post.query.where { created_at > 1.week.ago }
active_users = User.query.where(active: true)

Lustra::SQL.select
  .with_cte({recent_posts: recent_posts, active_users: active_users})
  .from(:recent_posts)
```

The CTE body can be another `SelectBuilder` or a raw SQL string.

## CTEs with Model Collections

Model collections are select builders, so they can be used as CTE bodies.

When querying a model from a CTE instead of its normal table, clear the default `FROM` and replace it with the CTE name:

```crystal
tagged_users = User.query
  .join(:posts) { posts.user_id == users.id }
  .join(:post_tags) { post_tags.post_id == posts.id }
  .join(:tags) { tags.id == post_tags.tag_id }
  .select("users.*, COUNT(DISTINCT tags.id) AS tag_count")
  .group_by("users.id")

users_with_multiple_tags = User.query
  .with_cte("tagged_users", tagged_users)
  .clear_from
  .from(:tagged_users)
  .where { tag_count > 1 }
```

This returns `User` instances built from the CTE rows.

## Recursive CTEs

Lustra currently documents `with_cte`, which emits regular `WITH` clauses. There is no separate `with_recursive` helper.

If you need `WITH RECURSIVE`, use raw SQL through the lower-level SQL execution APIs or keep that query as explicit SQL.

## Window Functions

PostgreSQL window functions can be written directly in selected SQL expressions:

```crystal
Employee.query
  .select(
    "employees.*",
    "SUM(salary) OVER (PARTITION BY department_id) AS department_salary"
  )
  .each(fetch_columns: true) do |employee|
    puts employee["department_salary"]
  end
```

This works well when the window expression is local to a single selected field.

## Named Windows

The lower-level select builder also exposes `window` for named window declarations:

```crystal
Lustra::SQL
  .select("SUM(salary) OVER w", "AVG(salary) OVER w")
  .from("employee_salaries")
  .window(w: "(PARTITION BY department_id ORDER BY salary DESC)")
```

Use named windows when several selected expressions share the same window definition.

