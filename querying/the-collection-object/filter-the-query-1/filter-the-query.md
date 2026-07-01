# Filter the Query - The Expression Engine

Collections represent SQL `SELECT` queries. Lustra's expression engine lets you write SQL conditions with Crystal syntax while still generating PostgreSQL SQL.

## Tuple and Hash Conditions

For simple equality checks, use named arguments, a `NamedTuple`, or a hash:

```crystal
User.query.where(first_name: "Richard")
User.query.where({first_name: "Richard", active: true})
```

Tuple/hash conditions also handle arrays, ranges, nil, and subqueries:

```crystal
User.query.where(id: [1, 2, 3])
User.query.where(created_at: 1.week.ago..Time.utc)
User.query.where(deleted_at: nil)

subquery = Post.query.select("user_id").where(published: true)
User.query.where(id: subquery)
```

## Template Conditions

Use templates when a small SQL fragment is clearer. Placeholder values are escaped:

```crystal
User.query.where("first_name = ? OR last_name = ?", "Richard", "Richard")
User.query.where("first_name = :name OR last_name = :name", name: "Richard")
```

Avoid interpolating user input directly into the SQL string.

## Expression Blocks

Use an expression block for richer conditions:

```crystal
User.query.where { (first_name == "Richard") | (last_name == "Richard") }
User.query.where { created_at.in?(1.week.ago..Time.utc) }
User.query.where { posts_count.between(10, 20) }
User.query.where { email =~ /@example\.com$/i }
```

Common operators:

| Crystal | SQL meaning |
| :--- | :--- |
| `==` / `!=` | equality / inequality, including `IS NULL` and `IS NOT NULL` for nil |
| `<`, `<=`, `>`, `>=` | comparison |
| `=~`, `!~` | PostgreSQL regex match / non-match |
| `&`, `\|` | `AND`, `OR` |
| `~condition` or `not(condition)` | `NOT` |
| `in?(array)` | `IN (...)` |
| `in?(range)` | range comparison |
| `in?(subquery)` | `IN (SELECT ...)` |
| `between(a, b)` | `BETWEEN a AND b` |

Because Crystal does not let libraries redefine `&&` and `||`, use `&` and `|` for SQL `AND` and `OR`.

Always parenthesize each side of `&` and `|`:

```crystal
User.query.where { (active == true) & (posts_count > 0) }
User.query.where { (role == "admin") | (role == "owner") }
```

## Local Variables and Column Names

The expression engine uses `method_missing` to turn unknown names into SQL variables. If a local variable has the same name as a column, Crystal resolves the local variable first:

```crystal
def find_user(id)
  User.query.where { id == id } # wrong: both sides are the local variable
end
```

Use `var` to force a SQL column:

```crystal
def find_user(id)
  User.query.where { var("id") == id }
end
```

`var` quotes each identifier part:

```crystal
User.query.where { var("users", "id") == 1 }
# "users"."id" = 1
```

## Raw SQL

Use `raw` for trusted SQL fragments that cannot be expressed through the normal API:

```crystal
User.query.where { raw("LOWER(email)") == "admin@example.com" }
User.query.where { raw("COUNT(*)") > 5 }
```

`raw` pastes SQL into the query. Prefer placeholders for values:

```crystal
User.query.where { raw("LOWER(email) = ?", email.downcase) }
User.query.where { raw("LOWER(email) = :email", email: email.downcase) }
```

{% hint style="danger" %}
Do not build `raw` SQL by interpolating untrusted input. Use placeholders or regular expression/tuple conditions instead.
{% endhint %}

## Custom Operators

Use `op` for PostgreSQL operators that do not map cleanly to Crystal operators:

```crystal
Event.query.where { op(payload, raw("'source'"), "?") }
```

For JSONB-specific querying, prefer Lustra's JSONB helpers where possible.

## Negation and OR

`where.not` wraps a condition in `NOT`:

```crystal
User.query.where.not(active: true)
User.query.where.not { deleted_at == nil }
```

`or` combines the current condition with a new condition:

```crystal
User.query.where(active: true).or(role: "admin")
User.query.where { posts_count > 10 }.or { role == "owner" }
```
