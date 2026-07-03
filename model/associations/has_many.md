# `has_many`

`has_many` is the inverse of a `belongs_to` relation. The related model stores the foreign key.

Using the author/book schema from [`belongs_to`](belongs_to.md):

```crystal
class Author
  include Lustra::Model

  primary_key
  column name : String

  has_many books : Book
end

class Book
  include Lustra::Model

  primary_key
  column title : String
  column published_at : Time?

  belongs_to author : Author
end
```

`Author#books` returns a `Book::Collection`:

```crystal
author = Author.query.find_by! { name == "Ada Lovelace" }

author.books.each do |book|
  puts book.title
end
```

Because the relation returns a collection, you can refine it before fetching:

```crystal
author.books
  .where { published_at != nil }
  .order_by(published_at: :desc)
  .each do |book|
    puts book.title
  end
```

## Adding Records

You can append a record to a `has_many` relation:

```crystal
author.books << Book.new({title: "A good book"})
```

Lustra sets the relation foreign key and saves the appended record.

## Autosave

By default, `build` creates an unsaved child record and does not save it when
the parent is saved.

```crystal
class Author
  include Lustra::Model

  has_many books : Book
end

author = Author.new({name: "Ada Lovelace"})
author.books.build({title: "Draft"})
author.save!

author.books.count # => 0
```

Use `autosave: true` when built child records should be saved with the parent.

```crystal
class Author
  include Lustra::Model

  has_many books : Book, autosave: true
end

author = Author.new({name: "Ada Lovelace"})
author.books.build({title: "Draft"})
author.save!

author.books.count # => 1
```

Autosave applies to records built through the association collection. Appending
with `<<` still saves the appended record immediately.

## Eager Loading

Collections get a generated `with_books` helper:

```crystal
Author.query.with_books.each do |author|
  author.books.each do |book|
    puts book.title
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
