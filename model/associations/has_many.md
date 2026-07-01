# `has_many`

`has_many` is the inverse of a `belongs_to` relation. The related model stores the foreign key.

Using the category/post schema from [`belongs_to`](belongs_to.md):

```crystal
class Category
  include Lustra::Model

  primary_key
  column name : String

  has_many posts : Post
end

class Post
  include Lustra::Model

  primary_key
  column name : String

  belongs_to category : Category
end
```

`Category#posts` returns a `Post::Collection`:

```crystal
category = Category.query.find_by! { name == "Technology" }

category.posts.each do |post|
  puts post.name
end
```

Because the relation returns a collection, you can refine it before fetching:

```crystal
category.posts
  .where { name =~ /^[0-9]/i }
  .order_by(name: "ASC")
  .each do |post|
    puts post.name
  end
```

## Adding Records

You can append a record to a `has_many` relation:

```crystal
category.posts << Post.new({name: "A good post"})
```

Lustra sets the relation foreign key and saves the appended record.

## Eager Loading

Collections get a generated `with_posts` helper:

```crystal
Category.query.with_posts.each do |category|
  category.posts.each do |post|
    puts post.name
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
