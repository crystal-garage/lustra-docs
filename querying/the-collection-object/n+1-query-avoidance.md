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
User.query.with_posts(&.with_category).to_a
# users query
# posts query
# categories query
```

Use eager loading when you will actually read the association for many parent records. Avoid it when only a few records need the association.

## Refining Eager-Loaded Associations

Pass a block to refine the association query:

```crystal
User.query.with_posts do |posts|
  posts.where(published: true).order_by(created_at: "DESC")
end
```

You can nest eager loading:

```crystal
User.query.with_posts do |posts|
  posts.where(published: true).with_category
end
```

The shorthand form is useful with scopes:

```crystal
User.query.with_posts(&.published.with_category)
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

