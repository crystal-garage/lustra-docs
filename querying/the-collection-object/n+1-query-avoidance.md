# Eager Loading

Eager loading helps avoid N+1 queries when reading associations for many records.

Without eager loading:

```crystal
Post.query.each do |post|
  puts post.user.email
end
```

This can run one query for posts, then one query per post for `user`.

With eager loading:

```crystal
Post.query.with_user.each do |post|
  puts post.user.email
end
```

Lustra loads the related users in an additional query and stores them in the collection's association cache.

## `with_*` Helpers

Association macros generate `with_*` helpers on collections:

```crystal
User.query.with_posts
Post.query.with_user
User.query.with_profile
Post.query.with_tags
```

The helper name follows the association name:

```crystal
class User
  include Lustra::Model

  has_many posts : Post
end

User.query.with_posts
```

## When Queries Run

Calling `with_posts` does not immediately execute SQL. It registers work that runs just before the parent query is fetched.

```crystal
query = User.query.with_posts # no SQL yet
users = query.to_a            # parent query and eager-loading query run here
```

The eager-loaded records are cached on the returned models. Later association calls use that cache:

```crystal
users.each do |user|
  user.posts.each do |post| # uses eager-loaded cache
    puts post.title
  end
end
```

The cache belongs to the collection/model instances involved in that fetch. It is not a global cache.

Eager-loading work also belongs to the collection on which `with_*` was called.
If you need independent query branches, duplicate the base query first and add
eager loading to each branch:

```crystal
base = User.query.where(active: true)
admins = base.dup.where(role: "admin").with_posts
editors = base.dup.where(role: "editor").with_posts
```

Do not call `dup` after `with_*` has already attached eager-loading work.

## Cost

Each `with_*` helper normally adds one extra query per association level.

```crystal
User.query.with_posts.to_a
# roughly:
# SELECT * FROM posts WHERE user_id IN (...)
# SELECT * FROM users
```

Nested eager loading adds more queries:

```crystal
User.query.with_posts(&.with_tags).to_a
# users query
# posts query
# tags query
```

Use eager loading when you will actually read the association for many parent records. Avoid it when only a few records need the association.

## Refining Eager-Loaded Associations

Pass a block to refine the association query:

```crystal
User.query.with_posts do |posts|
  posts.where(published: true).order_by(created_at: :desc)
end
```

You can nest eager loading:

```crystal
User.query.with_posts do |posts|
  posts.where(published: true).with_tags
end
```

The shorthand form is useful with scopes:

```crystal
User.query.with_posts(&.published.with_tags)
```

## Custom Fields

Some `with_*` helpers accept `fetch_columns` for related models:

```crystal
Post.query.with_user(fetch_columns: true) do |users|
  users.select("users.*", "LOWER(users.email) AS normalized_email")
end
```

Use this only when you need custom selected fields from the eager-loaded relation.

## Caveats

`with_*` uses the parent query as a subquery when loading related records. If the parent query is expensive, the eager-loading query can also be expensive.

For large or complex parent queries, inspect the generated SQL and PostgreSQL query plan.
