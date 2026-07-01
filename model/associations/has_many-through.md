# `has_many through`

`has_many through` connects two models through a join model/table.

Example schema:

```sql
CREATE TABLE posts (
  id bigserial PRIMARY KEY,
  name text NOT NULL
);

CREATE TABLE tags (
  id bigserial PRIMARY KEY,
  name text NOT NULL
);

CREATE TABLE post_tags (
  post_id bigint NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  tag_id bigint NOT NULL REFERENCES tags(id) ON DELETE CASCADE
);
```

Define a join model for the join table:

```crystal
class PostTag
  include Lustra::Model

  self.table = "post_tags"

  belongs_to post : Post, foreign_key_type: Int64?
  belongs_to tag : Tag, foreign_key_type: Int64?
end
```

Then connect the main models through it:

```crystal
class Post
  include Lustra::Model

  primary_key
  column name : String

  has_many tags : Tag, through: PostTag
end

class Tag
  include Lustra::Model

  primary_key
  column name : String

  has_many posts : Post, through: PostTag
end
```

Now `post.tags` returns a `Tag::Collection`:

```crystal
post = Post.query.first!

post.tags.each do |tag|
  puts tag.name
end
```

## Adding and Removing Links

Append creates the target record if needed and inserts the join row when it does not already exist:

```crystal
post = Post.query.first!
tag = Tag.query.find_or_create({name: "Technology"}) { }

post.tags << tag
```

Unlink deletes the join row:

```crystal
post = Post.query.first!
tag = post.tags.where(name: "Technology").first!

post.tags.unlink(tag)
```

Call `unlink` on the relation collection (`post.tags`), not on a plain `Tag.query`, so Lustra knows which parent and through table to use.

## `DISTINCT` and Advanced Queries

`has_many through` queries use `DISTINCT` to avoid returning the same target
record more than once when the join table can match multiple rows.

For normal association reads, keep the default:

```crystal
post.tags.each do |tag|
  puts tag.name
end
```

For advanced queries, you may need to remove that `DISTINCT` clause before
adding custom select fields, grouping, or ordering.

```crystal
repositories = tag
  .repositories
  .clear_distinct
  .select(
    "repositories.*",
    "(select COUNT(*) from relationships r WHERE r.dependency_id = repositories.id) AS dependents_count"
  )
  .group_by("repositories.id")
  .order_by("dependents_count", :desc)
```

Use `clear_distinct` deliberately. If the join can produce duplicate target
rows, removing `DISTINCT` allows those duplicates to appear unless your query
handles them with grouping or another condition.

## Eager Loading

`has_many through` also generates a `with_*` helper:

```crystal
Post.query.with_tags.each do |post|
  post.tags.each do |tag|
    puts tag.name
  end
end
```

## Options

```crystal
has_many relation_name : RelationType,
  through: JoinModel,
  foreign_key: "target_id",
  own_key: "owner_id",
  autosave: true
```

| Option | Description | Default |
| :--- | :--- | :--- |
| `through` | Join model used to connect the two tables. | required |
| `foreign_key` | Join-table column pointing to the target model. | target table singularized + `_id` |
| `own_key` | Join-table column pointing to the current model. | current table singularized + `_id` |
| `autosave` | Saves built association records when the parent is saved. | `false` |

The join model does not need a primary key for simple join-table usage, but it should still map the join table accurately.
