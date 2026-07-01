# Custom Selected Fields

Sometimes a query selects fields that do not belong to the model itself: aggregate values, computed fields, or columns from joined tables.

Use `fetch_columns: true` to keep those values in the model's `attributes` hash.

```crystal
posts = Post.query
  .select(
    "posts.*",
    "COUNT(comments.id) AS comments_count"
  )
  .left_join(:comments) { comments.post_id == posts.id }
  .group_by("posts.id")

posts.each(fetch_columns: true) do |post|
  puts "#{post.title}: #{post.attributes["comments_count"]}"
end
```

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
