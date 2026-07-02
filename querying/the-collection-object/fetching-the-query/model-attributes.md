# Custom Selected Fields

Sometimes a query selects fields that are not model columns: aggregate values,
computed fields, or columns from joined tables.

By default, Lustra initializes the model columns it knows about. Use
`fetch_columns: true` when you also want to keep custom selected fields in the
model's `attributes` hash.

```crystal
posts = Post.query
  .select("posts.*", "COUNT(post_tags.id) AS tags_count")
  .left_join(:post_tags) { post_tags.post_id == posts.id }
  .group_by("posts.id")

posts.each(fetch_columns: true) do |post|
  puts post.title
  puts post.attributes["tags_count"]
end
```

`post.title` uses the generated model accessor. `tags_count` is not a model
column, so it is read from `attributes`.

You can also use `[]` and `[]?` on the model:

```crystal
post["tags_count"]  # raises if the key is missing
post["tags_count"]? # nil if the key is missing
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
posts = Post.query
  .with_count(:tags, alias_name: "tags_count")

posts.each(fetch_columns: true) do |post|
  puts post.title
  puts post["tags_count"]
end
```

If you only need raw rows and not models, use `fetch` instead:

```crystal
Post.query
  .select("posts.id", "COUNT(post_tags.id) AS tags_count")
  .left_join(:post_tags) { post_tags.post_id == posts.id }
  .group_by("posts.id")
  .fetch do |row|
    puts row["tags_count"]
  end
```
