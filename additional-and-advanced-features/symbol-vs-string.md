# Symbol vs String

Many query-building methods accept both symbols and strings.

As a rule of thumb:

* `Symbol` values are treated as SQL identifiers and escaped with double quotes.
* `String` values are treated as explicit SQL fragments or names and are often used as-is.

Example:

```crystal
User.query.join(:orders) { order_id == orders.id }
# INNER JOIN "orders" ...

User.query.join("orders") { order_id == orders.id }
# INNER JOIN orders ...
```

Use symbols for ordinary table or column identifiers when you want Lustra to quote them.

Use strings when you intentionally need an explicit SQL fragment, an alias, a function call, or a pre-qualified name.

For expression blocks, `var("table", "column")` is the safest way to build quoted identifiers:

```crystal
User.query.where { var("users", "email") == "admin@example.com" }
```

For raw SQL fragments, use placeholders for values:

```crystal
User.query.where("LOWER(email) = ?", "admin@example.com")
```
