# Ordering, Grouping, and Having

## Ordering

Use `order_by` to add `ORDER BY` clauses:

```crystal
User.query.order_by(last_name: "ASC", first_name: "ASC").each do |user|
  puts "#{user.first_name} #{user.last_name}"
end
```

Symbols are escaped as SQL identifiers:

```crystal
User.query.order_by(:created_at, :desc)
```

Strings are used as SQL expressions:

```crystal
User.query.order_by("LOWER(email)", :asc)
```

You can control null ordering:

```crystal
User.query.order_by(last_login_at: {:desc, :nulls_last})
```

Use `reverse_order_by` to flip directions:

```crystal
query = User.query.order_by(created_at: :desc)
query.reverse_order_by
```

Use `clear_order_bys` to remove ordering:

```crystal
query.clear_order_bys
```

`order_by`, `reverse_order_by`, and `clear_order_bys` mutate the collection.

### `reverse_order_by`

`reverse_order_by` flips every existing order direction and also swaps null sort
behavior. It keeps all current order keys and only changes `ASC/DESC` and
`NULLS FIRST/LAST`.

```crystal
query = User.query.order_by(created_at: :desc, last_login_at: {:asc, :nulls_last})
query.reverse_order_by
# => ORDER BY "created_at" ASC, "last_login_at" DESC NULLS FIRST
```

### `in_order_of`

`in_order_of` orders rows by a custom list of values for a column.

```crystal
Post.query.in_order_of(:status, ["open", "waiting", "closed"]).each do |post|
  puts post.status
end
```

It appends a `CASE` expression to the current `ORDER BY` list and keeps items
not in the list after those values.

```crystal
Post.query
  .where(published: true)
  .order_by(priority: :desc)
  .in_order_of(:status, ["high", "medium", "low"])
```

Use it with symbol columns or raw SQL expressions:

```crystal
Post.query.in_order_of("\"posts\".\"status\"", ["high", "low"])
```

### `NULLS FIRST` and `NULLS LAST`

Use the second tuple element to control null sorting.

```crystal
User.query.order_by(last_login_at: {:desc, :nulls_last})
User.query.order_by(created_at: {:asc, :nulls_first})
```

## Custom Value Ordering

Use `in_order_of` to sort rows by a specific list of values:

```crystal
Post.query.in_order_of(:status, ["draft", "published", "archived"])
```

Rows whose value is not in the list sort after the listed values.

## Grouping

Use `group_by` to add a `GROUP BY` clause:

```crystal
Post.query
  .select("user_id", "COUNT(*) AS posts_count")
  .group_by(:user_id)
```

Symbols are escaped. Strings are used as SQL expressions:

```crystal
Post.query.group_by("DATE(created_at)")
```

Use `clear_group_bys` to remove grouping.

## Having

Use `having` to filter grouped rows:

```crystal
Post.query
  .select("user_id", "COUNT(*) AS posts_count")
  .group_by(:user_id)
  .having { raw("COUNT(*)") > 5 }
```

`having` supports the same condition styles as `where`:

```crystal
Post.query.group_by(:user_id).having("COUNT(*) > ?", 5)
Post.query.group_by(:user_id).having("COUNT(*) > :count", count: 5)
```

Use `or_having` to combine grouped conditions with `OR`:

```crystal
Post.query
  .group_by(:user_id)
  .having { raw("COUNT(*)") > 5 }
  .or_having { raw("MAX(created_at)") > 1.week.ago }
```

Use `clear_havings` to remove `HAVING` clauses.
