# `belongs_to`

`belongs_to` is used when the current model stores the foreign key.

Example schema:

```sql
CREATE TABLE users (
  id bigserial PRIMARY KEY,
  email text NOT NULL,
  active boolean NOT NULL DEFAULT TRUE
);

CREATE TABLE posts (
  id bigserial PRIMARY KEY,
  title text NOT NULL,
  published boolean NOT NULL DEFAULT FALSE,
  user_id bigint NOT NULL REFERENCES users(id)
);
```

`posts.user_id` points to `users.id`, so `Post` belongs to `User`:

```crystal
class User
  include Lustra::Model

  primary_key
  column email : String
  column active : Bool = true

  has_many posts : Post
end

class Post
  include Lustra::Model

  primary_key
  column title : String
  column published : Bool = false

  belongs_to user : User
end
```

The `belongs_to` macro declares the `user_id` column for you. You do not need to declare it separately unless you want a custom setup.

Use the association like this:

```crystal
post = Post.query.first!
puts post.user.email
```

Assigning a persisted parent updates the foreign key:

```crystal
user = User.query.find_by! { email == "ada@example.com" }
post.user = user
post.save!
```

## Counter Cache

Use `counter_cache` when the parent model stores the number of child records.

```crystal
class User
  include Lustra::Model

  primary_key
  column email : String
  column posts_count : Int64 = 0

  has_many posts : Post
end

class Post
  include Lustra::Model

  primary_key
  column title : String

  belongs_to user : User, counter_cache: true
end
```

With `counter_cache: true`, Lustra uses the conventional counter column name:
the child table name plus `_count`. For `Post.table == "posts"`, the parent
column is `posts_count`.

```crystal
user = User.create!(email: "ada@example.com")
Post.create!(title: "First", user: user)

user.reload
user.posts_count # => 1
```

The counter is incremented after child creation and decremented after child
destroy:

```crystal
post = Post.create!(title: "Second", user: user)
post.destroy
```

Use a custom counter column by passing the column name:

```crystal
class Post
  include Lustra::Model

  belongs_to user : User, counter_cache: :published_posts_count
end
```

If counters become stale, reset them from the parent model class:

```crystal
User.reset_counters(user.id, Post)
```

Or from a parent record:

```crystal
user.reset_counters(Post)
```

Counter caches rely on model lifecycle callbacks. Direct SQL, `delete`,
`delete_all`, and other lifecycle-bypassing operations do not update counter
columns. Use `reset_counters` after those operations if the counter must remain
accurate.

## Nilable Associations

Use a nilable relation type when the foreign key can be null:

```crystal
class Post
  include Lustra::Model

  primary_key
  belongs_to user : User?, foreign_key_type: Int64?
end
```

This generates `user : User?` and `user! : User`.

## Options

```crystal
belongs_to relation_name : RelationType,
  foreign_key: "column_name",
  primary: true,
  foreign_key_type: Int64,
  touch: true,
  counter_cache: true
```

| Option | Description | Default |
| :--- | :--- | :--- |
| `foreign_key` | Column stored on the current model. | `[relation_name]_id` |
| `primary` | Marks the generated foreign key column as the primary key. | `false` |
| `foreign_key_type` | Crystal type for the generated foreign key column. Use a nilable type for optional associations. | `Int64` |
| `touch` | Touches the parent when the child changes. Use `true` for `updated_at` or a symbol/string for a specific timestamp column. | `nil` |
| `counter_cache` | Updates a counter column on the parent. Use `true` for the conventional counter name or pass a specific column name. | `nil` |
