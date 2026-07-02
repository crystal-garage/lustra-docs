# Find, First, Last, Offset, and Limit

These helpers fetch one record, a small slice of records, or refine pagination.

## `find` and `find!`

`find` looks up records by primary key.

```crystal
post = Post.query.find(1234)  # Post?
post = Post.query.find!(1234) # Post or raises
```

`find` returns `nil` when no row exists. `find!` raises `Lustra::SQL::RecordNotFoundError`.

You can also pass an array of primary keys:

```crystal
posts = Post.query.find([1, 2, 3])  # Array(Post)
posts = Post.query.find!([1, 2, 3]) # raises if any id is missing
```

## `find_by` and `find_by!`

`find_by` applies a condition and returns the first matching record:

```crystal
user = User.query.find_by(email: "user@example.com")
user = User.query.find_by { (active == true) & (email == "user@example.com") }
```

`find_by` returns `nil` when no row exists. `find_by!` raises:

```crystal
user = User.query.find_by!(email: "user@example.com")
```

Model classes expose shortcuts:

```crystal
User.find_by(email: "user@example.com")
User.find_by!(email: "user@example.com")
```

## `first` and `first!`

`first` returns the first matching record or `nil`:

```crystal
post = Post.query.order_by(created_at: :desc).first
```

`first!` returns the record or raises `Lustra::SQL::RecordNotFoundError`:

```crystal
post = Post.query.order_by(created_at: :desc).first!
```

When no explicit order is set, Lustra applies primary-key ordering before fetching the first row.

## `last` and `last!`

`last` returns the last record according to the current ordering:

```crystal
post = Post.query.order_by(created_at: :desc).last
```

`last!` raises when no row exists:

```crystal
post = Post.query.order_by(created_at: :desc).last!
```

`last` works by reversing the current order and fetching one row. If no order is set, Lustra uses primary-key ordering.

## `limit` and `offset`

`limit` and `offset` refine the collection:

```crystal
posts = Post.query
  .order_by(id: :asc)
  .limit(20)
  .offset(40)
```

This fetches up to 20 records after skipping 40 records.

Because collections are mutable, `limit` and `offset` change the collection they are called on. Use `dup` if you need to keep a reusable base query:

```crystal
base = Post.query.order_by(id: :asc)
page_1 = base.dup.limit(20).offset(0)
page_2 = base.dup.limit(20).offset(20)
```

## Array-Style Access

`[]` and `[]?` resolve the query. They return records, not reusable collections.

```crystal
post = Post.query.order_by(id: :asc)[10]  # Post or raises
post = Post.query.order_by(id: :asc)[10]? # Post?
```

`[]` with a range returns an array:

```crystal
posts = Post.query.order_by(id: :asc)[10..20] # Array(Post)
```

Internally, array-style access applies `offset` and `limit` to the collection. If you need to keep the original collection unchanged, call it on a duplicate:

```crystal
posts = Post.query.order_by(id: :asc)
slice = posts.dup[10..20]
```

For general array indexing after loading records, use `to_a`:

```crystal
posts = Post.query.order_by(id: :asc).to_a
post = posts[10]?
```
