# Each, Map, Fetch, and To Array

`Collection` includes `Enumerable(T)`, so calling enumeration methods resolves the query.

## `each`

`each` builds model instances:

```crystal
Post.query.where(user_id: 1).each do |post|
  puts post.title
end
```

Pass `fetch_columns: true` when you also need custom selected fields in `attributes`:

```crystal
Post.query
  .select("posts.*", "COUNT(post_tags.id) AS tags_count")
  .left_join(:post_tags) { post_tags.post_id == posts.id }
  .group_by("posts.id")
  .each(fetch_columns: true) do |post|
    puts post.attributes["tags_count"]
  end
```

## `map`

`map` also builds model instances, then applies the block:

```crystal
emails = User.query.where(active: true).map(&.email)
```

Like `each`, it accepts `fetch_columns`:

```crystal
labels = User.query
  .select("users.*", "LOWER(email) AS normalized_email")
  .map(fetch_columns: true) { |user| user["normalized_email"].as(String) }
```

## `pluck_col` and `pluck`

Use `pluck_col` when you only need one column and do not need model instances:

```crystal
names = User.query
  .where(active: true)
  .order_by(id: :asc)
  .pluck_col("email")
```

Without a type argument, `pluck_col` returns `Array(Lustra::SQL::Any)`. Pass a
type when you want a typed array:

```crystal
emails = User.query.pluck_col("email", String)
ids = User.query.pluck_col(:id, Int64)
```

Use `pluck` for multiple columns:

```crystal
posts = Post.query.pluck("title", "published")

posts.each do |(title, published)|
  puts "#{title}: #{published}"
end
```

Named `pluck` returns typed tuples:

```crystal
users = User.query.pluck(
  "email": String,
  "LOWER(email)": String
)
```

`pluck` and `pluck_col` execute a copied query with a new `SELECT` list, so they
do not instantiate models. Use them for simple column extraction. Use `each`,
`map`, or `fetch` when you need models or full raw rows.

String fields are SQL fragments. Use symbols for simple column names when
possible, and do not interpolate untrusted input into string fragments.

## `to_a`

`to_a` loads all matching model records into an array:

```crystal
users = User.query.where(active: true).to_a
```

Use this when you intentionally want array behavior:

```crystal
users = User.query.order_by(id: "ASC").to_a
user = users[10]?
```

`to_a(fetch_columns: true)` keeps custom selected fields in each model's `attributes` hash.

## `fetch`

`fetch` comes from the lower-level SQL query builder. It yields `Hash(String, Lustra::SQL::Any)` rows instead of model instances:

```crystal
Post.query.where(user_id: 1).fetch do |row|
  puts "#{row["id"]} - #{row["title"]}"
end
```

Use `fetch` when you need raw SQL rows or want to avoid model construction overhead.

`fetch(fetch_all: true)` loads the result rows before yielding them. This is useful when the block itself needs to run more SQL, because the original result set is closed before the block runs.
