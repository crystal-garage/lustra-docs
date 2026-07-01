# Custom Selected Fields

Sometimes a query selects fields that are not model columns: aggregate values,
computed fields, or columns from joined tables.

By default, Lustra initializes the model columns it knows about. Use
`fetch_columns: true` when you also want to keep custom selected fields in the
model's `attributes` hash.

```crystal
posts = Post.query
  .select(
    "posts.*",
    "COUNT(comments.id) AS comments_count"
  )
  .left_join(:comments) { comments.post_id == posts.id }
  .group_by("posts.id")

posts.each(fetch_columns: true) do |post|
  puts post.title
  puts post.attributes["comments_count"]
end
```

`post.title` uses the generated model accessor. `comments_count` is not a model
column, so it is read from `attributes`.

You can also use `[]` and `[]?` on the model:

```crystal
post["comments_count"]  # raises if the key is missing
post["comments_count"]? # nil if the key is missing
```

`fetch_columns: true` is available on model-fetching helpers such as:

* `each`
* `map`
* `to_a`
* `first` / `first!`
* `last` / `last!`
* `find_by` / `find_by!`
* `[]`, `[]?`, and range access
* `each_with_cursor`

{% hint style="info" %}
`fetch_columns: true` stores every returned SQL field in `attributes`, so it has extra allocation cost. Leave it disabled unless you need custom selected fields.
{% endhint %}

This pattern is useful for reusable computed fields:

```crystal
repositories = Repository.query
  .select(
    "repositories.*",
    "(select COUNT(*) from relationships r WHERE r.dependency_id = repositories.id) AS dependents_count",
    "(select COUNT(*) from relationships r WHERE r.master_id = repositories.id) AS dependencies_count"
  )

repositories.each(fetch_columns: true) do |repository|
  puts repository.name
  puts repository["dependents_count"]
  puts repository["dependencies_count"]
end
```

If you only need raw rows and not models, use `fetch` instead:

```crystal
Post.query
  .select("posts.id", "COUNT(comments.id) AS comments_count")
  .left_join(:comments) { comments.post_id == posts.id }
  .group_by("posts.id")
  .fetch do |row|
    puts row["comments_count"]
  end
```
