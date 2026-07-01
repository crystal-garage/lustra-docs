---
description: Lustra provides several ways to filter and refine SQL queries.
---

# Filter the Query

Filtering is usually done with `where`, the expression engine, or the lower-level SQL builder.

Use the simplest form that fits the query:

```crystal
User.query.where(active: true)
User.query.where { (first_name == "Ada") | (last_name == "Lovelace") }
User.query.where("created_at >= ?", 1.week.ago)
```

Each call to `where` adds another condition. Multiple `where` calls are combined with `AND` unless you explicitly use `or`.
