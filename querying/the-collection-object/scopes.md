# Scopes

Scopes package commonly used query fragments into named methods.

```crystal
class Post
  include Lustra::Model

  column id : Int64, primary: true, presence: false
  column published : Bool
  column created_at : Time

  scope("published") { where(published: true) }
  scope("recent") { where { created_at > 7.days.ago }.order_by(created_at: :desc) }
end
```

Scopes are available on the model class:

```crystal
Post.published
Post.recent
```

They are also available on collections, so they can be chained:

```crystal
Post.published.recent
Post.published.recent.first
```

## Scope Arguments

Scopes can receive arguments:

```crystal
scope("with_title") { |title| where(title: title) }
```

Scopes can also receive splat arguments:

```crystal
scope("with_ids") { |*ids| where { id.in?(ids) } }
```

Use scopes for reusable filtering, ordering, or query composition. Keep them small and predictable.

## Computed Fields

Scopes can also add computed SQL fields that are reused by several queries.

```crystal
class Post
  include Lustra::Model

  column id : Int64, primary: true, presence: false
  column title : String

  scope("with_tag_count") do
    with_count(:tags, alias_name: "tags_count")
  end
end
```

The scope can be chained with filters, ordering, and pagination:

```crystal
posts = Post
  .query
  .with_tag_count
  .where(published: true)
  .order_by("tags_count", :desc)
  .limit(20)
```

When a scope selects fields that are not model columns, fetch them with
`fetch_columns: true` and read them from `attributes` or `[]`.

```crystal
posts.each(fetch_columns: true) do |post|
  puts post.title
  puts post["tags_count"]
end
```

## Mutability

Scope blocks run against a collection and mutate that collection like other query-refinement methods.

```crystal
posts = Post.query
published = posts.published

# posts and published refer to the same refined collection.
```

Use `dup` if you need to preserve a base query:

```crystal
posts = Post.query
published = posts.dup.published
recent = posts.dup.recent
```

## Default Scopes

`default_scope` automatically applies a query fragment to all model queries.

This is useful for soft deletes or tenant filters:

```crystal
class Post
  include Lustra::Model

  column id : Int64, primary: true, presence: false
  column deleted_at : Time?

  default_scope { where { deleted_at == nil } }
end
```

Now regular queries include the default scope:

```crystal
Post.query.count
Post.find(1)
Post.query.where(title: "Hello")
```

## `unscoped`

Use `unscoped` to start a query without the default scope:

```crystal
Post.query.unscoped.count
Post.query.unscoped.where { deleted_at != nil }
```

`unscoped` returns a fresh collection without the default scope applied.

## Default Scope Caveat

Default scopes are implicit. They affect `query`, scopes, `find`, `find_by`, counts, and association queries built from the model.

Use them sparingly and document why they exist. Prefer explicit scopes when hidden filtering would make code harder to reason about.
