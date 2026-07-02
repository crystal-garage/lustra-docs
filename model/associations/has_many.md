# `has_many`

`has_many` is the inverse of a `belongs_to` relation. The related model stores the foreign key.

Using the user/post schema from [`belongs_to`](belongs_to.md):

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

`User#posts` returns a `Post::Collection`:

```crystal
user = User.query.find_by! { email == "ada@example.com" }

user.posts.each do |post|
  puts post.title
end
```

Because the relation returns a collection, you can refine it before fetching:

```crystal
user.posts
  .where(published: true)
  .order_by(created_at: :desc)
  .each do |post|
    puts post.title
  end
```

## Adding Records

You can append a record to a `has_many` relation:

```crystal
user.posts << Post.new({title: "A good post"})
```

Lustra sets the relation foreign key and saves the appended record.

## Autosave

By default, `build` creates an unsaved child record and does not save it when
the parent is saved.

```crystal
class User
  include Lustra::Model

  has_many posts : Post
end

user = User.new({email: "ada@example.com"})
user.posts.build({title: "Draft"})
user.save!

user.posts.count # => 0
```

Use `autosave: true` when built child records should be saved with the parent.

```crystal
class User
  include Lustra::Model

  has_many posts : Post, autosave: true
end

user = User.new({email: "ada@example.com"})
user.posts.build({title: "Draft"})
user.save!

user.posts.count # => 1
```

Autosave applies to records built through the association collection. Appending
with `<<` still saves the appended record immediately.

## Eager Loading

Collections get a generated `with_posts` helper:

```crystal
User.query.with_posts.each do |user|
  user.posts.each do |post|
    puts post.title
  end
end
```

The `with_*` helper runs an additional query and fills the relation cache to avoid N+1 queries.

## Options

```crystal
has_many relation_name : RelationType,
  foreign_key: "column_name",
  primary_key: "column_name",
  autosave: true
```

| Option | Description | Default |
| :--- | :--- | :--- |
| `foreign_key` | Column stored on the related model. | current table singularized + `_id` |
| `primary_key` | Column on the current model matched against the foreign key. | model primary key |
| `autosave` | Saves built child records when the parent is saved. | `false` |
