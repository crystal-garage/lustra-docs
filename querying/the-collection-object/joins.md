# Joins

Collections include Lustra's SQL select builder, so they can build joins directly.

## Manual Joins

Use `join`, `inner_join`, `left_join`, `right_join`, `full_outer_join`, or `cross_join`.

```crystal
Post.query
  .inner_join(:users) { users.id == posts.user_id }
  .where { users.active == true }
```

Symbols are escaped as SQL identifiers:

```crystal
Post.query.inner_join(:users) { users.id == posts.user_id }
```

Strings are used as SQL fragments:

```crystal
Post.query.inner_join("users u") { var("u", "id") == posts.user_id }
```

You can also pass a SQL condition string:

```crystal
Post.query.inner_join("users", "users.id = posts.user_id")
```

## Association Joins

When a symbol matches a model association, Lustra can infer the join condition:

```crystal
User.query.join(:posts)
Post.query.join(:user)
User.query.left_join(:posts)
```

Association joins work for:

* `belongs_to`
* `has_many`
* `has_one`
* `has_many through`

For `has_many through`, Lustra adds the join table and the target table.

If the symbol does not match an association, Lustra raises an error listing available associations. Use a string or a block join for plain table names:

```crystal
User.query.join("posts") { posts.user_id == users.id }
```

## Subquery and Lateral Joins

Join targets can be subqueries:

```crystal
latest_posts = Post.query
  .select("DISTINCT ON (user_id) *")
  .order_by("user_id ASC, created_at DESC")

User.query.left_join(latest_posts, lateral: true)
```

Use `lateral: true` when the join target should be a PostgreSQL `LATERAL` join.

## Filtering with Joins

Joined tables can be referenced in expression blocks:

```crystal
User.query
  .join(:posts)
  .where { posts.title =~ /crystal/i }
  .distinct("users.id")
```

Use `var("table", "column")` when you need explicit quoted identifiers:

```crystal
Post.query
  .inner_join("users u") { var("u", "id") == posts.user_id }
  .where { var("u", "active") == true }
```

## Join-Based Subqueries

Collections can be used as subqueries:

```crystal
def users_with_more_than_posts(count)
  User.query
    .select("users.id")
    .inner_join(:posts) { posts.user_id == users.id }
    .group_by("users.id")
    .having { raw("COUNT(*)") > count }
end

Post.query.where { user_id.in?(users_with_more_than_posts(10)) }
```

## Notes

Joining does not eager load associations. If you want to avoid N+1 queries while reading associations, use the generated `with_*` helpers described in [Eager Loading](n+1-query-avoidance.md).

Use joins when the association or table needs to affect filtering, ordering, grouping, or selected fields.

