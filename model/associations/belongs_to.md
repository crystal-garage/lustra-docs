# `belongs_to`

`belongs_to` is used when the current model stores the foreign key.

Example schema:

```sql
CREATE TABLE authors (
  id bigserial PRIMARY KEY,
  name text NOT NULL
);

CREATE TABLE books (
  id bigserial PRIMARY KEY,
  title text NOT NULL,
  published_at timestamp,
  author_id bigint NOT NULL REFERENCES authors(id)
);
```

`books.author_id` points to `authors.id`, so `Book` belongs to `Author`:

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

The `belongs_to` macro declares the `author_id` column for you. You do not need
to declare it separately unless you want a custom setup.

Use the association like this:

```crystal
book = Book.query.first!
puts book.author.name
```

Assigning a persisted parent updates the foreign key:

```crystal
author = Author.query.find_by! { name == "Ada Lovelace" }
book.author = author
book.save!
```

## Counter Cache

Use `counter_cache` when the parent model stores the number of child records.

```crystal
class Author
  include Lustra::Model

  primary_key
  column name : String
  column books_count : Int64 = 0

  has_many books : Book
end

class Book
  include Lustra::Model

  primary_key
  column title : String

  belongs_to author : Author, counter_cache: true
end
```

With `counter_cache: true`, Lustra uses the conventional counter column name:
the child table name plus `_count`. For `Book.table == "books"`, the parent
column is `books_count`.

```crystal
author = Author.create!(name: "Ada Lovelace")
Book.create!(title: "First", author: author)

author.reload
author.books_count # => 1
```

The counter is incremented after child creation and decremented after child
destroy:

```crystal
book = Book.create!(title: "Second", author: author)
book.destroy
```

Use a custom counter column by passing the column name:

```crystal
class Book
  include Lustra::Model

  belongs_to author : Author, counter_cache: :published_books_count
end
```

If counters become stale, reset them from the parent model class:

```crystal
Author.reset_counters(author.id, Book)
```

Or from a parent record:

```crystal
author.reset_counters(Book)
```

Counter caches rely on model lifecycle callbacks. Direct SQL, `delete`,
`delete_all`, and other lifecycle-bypassing operations do not update counter
columns. Use `reset_counters` after those operations if the counter must remain
accurate.

## Nilable Associations

Use a nilable relation type when the foreign key can be null:

```crystal
class Book
  include Lustra::Model

  primary_key
  belongs_to author : Author?, foreign_key_type: Int64?
end
```

This generates `author : Author?` and `author! : Author`.

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
