# Scopes

Scopes package commonly used query fragments into named methods.

```crystal
class Post
  include Lustra::Model

  column id : Int64, primary: true, presence: false
  column published : Bool
  column view_count : Int32
  column created_at : Time

  scope("published") { where(published: true) }
  scope("popular") { |min_views| where { view_count >= min_views } }
  scope("recent") { where { created_at > 7.days.ago }.order_by(created_at: :desc) }
end
```

Scopes are available on the model class:

```crystal
Post.published
Post.popular(100)
```

They are also available on collections, so they can be chained:

```crystal
Post.published.recent
Post.published.popular(50).first
```

## Scope Arguments

Scopes can receive arguments:

```crystal
scope("with_status") { |status| where(status: status) }
```

Scopes can also receive splat arguments:

```crystal
scope("with_ids") { |*ids| where { id.in?(ids) } }
```

Use scopes for reusable filtering, ordering, or query composition. Keep them small and predictable.

## Computed Fields

Scopes can also add computed SQL fields that are reused by several queries.

```crystal
class Repository
  include Lustra::Model

  column id : Int64, primary: true, presence: false
  column name : String

  scope("with_counts") do
    self
      .select(
        "repositories.*",
        "(select COUNT(*) from relationships r WHERE r.dependency_id = repositories.id) AS dependents_count",
        "(select COUNT(*) from relationships r WHERE r.master_id = repositories.id) AS dependencies_count"
      )
      .group_by("repositories.id")
  end
end
```

The scope can be chained with filters, ordering, and pagination:

```crystal
repositories = Repository
  .query
  .with_counts
  .where(ignore: false)
  .order_by("dependents_count", :desc)
  .limit(20)
```

When a scope selects fields that are not model columns, fetch them with
`fetch_columns: true` and read them from `attributes` or `[]`.

```crystal
repositories.each(fetch_columns: true) do |repository|
  puts repository.name
  puts repository["dependents_count"]
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
