# Model Basics

This guide introduces the core ideas behind Lustra models.

After reading it, you should understand:

* how a Lustra model maps to a PostgreSQL table
* how Lustra uses naming conventions
* how to declare columns and primary keys
* how to create, read, update, and delete records
* where validations, callbacks, migrations, associations, and querying fit

## What is a Lustra Model?

Lustra is an ORM for Crystal and PostgreSQL. A model class represents a database
table, and a model instance represents one row in that table.

```crystal
class Book
  include Lustra::Model

  primary_key
  column title : String
  column author_name : String
end
```

This model maps to a `books` table:

```sql
CREATE TABLE books (
  id bigserial PRIMARY KEY,
  title text NOT NULL,
  author_name text NOT NULL
);
```

Each `Book` object carries data from one row and the behavior you define on the
model class.

## Convention over Configuration

Lustra follows common Active Record style naming conventions:

| Crystal model | PostgreSQL table |
| :--- | :--- |
| `Book` | `books` |
| `BookOrder` | `book_orders` |
| `Catalog::Book` | `catalog_books` |

You can override the table name when your database uses another convention:

```crystal
class Book
  include Lustra::Model

  self.table = "library_books"

  primary_key
  column title : String
end
```

## Schema Conventions

Lustra does not inspect the database schema to generate model attributes at
runtime. Columns are declared in Crystal so the compiler can type-check your
model code.

Common conventions:

| Column | Purpose |
| :--- | :--- |
| `id` | Default primary key, usually declared with `primary_key`. |
| `created_at`, `updated_at` | Timestamp columns declared with `timestamps`. |
| `author_id`, `book_id` | Foreign keys used by associations. |
| `<name>_id`, `<name>_type` | Polymorphic association columns. |
| `<table>_count` | Counter cache columns, such as `books_count`. |

## Creating a Model

Start by including `Lustra::Model`, then declare the table columns:

```crystal
class Book
  include Lustra::Model

  primary_key
  column title : String
  column author_name : String
  column published_at : Time?
  timestamps
end
```

`primary_key` declares an auto-generated `id : Int64` primary key by default.
Nilable Crystal types, such as `Time?`, map to nullable columns.

See [Defining your model](column-types/README.md) for column options and custom
converters.

## CRUD: Reading and Writing Data

CRUD means create, read, update, and delete.

Create a new unsaved model with `new`:

```crystal
book = Book.new({
  title: "The Hobbit",
  author_name: "J. R. R. Tolkien",
})

book.save!
```

Create and save in one call with `create!`:

```crystal
book = Book.create!(
  title: "The Lord of the Rings",
  author_name: "J. R. R. Tolkien"
)
```

Read records through a collection query:

```crystal
book = Book.query.find!(1)

Book.query
  .where(author_name: "J. R. R. Tolkien")
  .order_by(title: :asc)
  .each do |book|
    puts book.title
  end
```

Update a record and save it:

```crystal
book.title = "The Hobbit, or There and Back Again"
book.save!
```

Or assign and save in one call:

```crystal
book.update!(title: "The Hobbit")
```

Remove a record with `destroy` when callbacks should run:

```crystal
book.destroy
```

Use `delete` only when you intentionally want to remove the row without running
destroy callbacks.

See [Persistence](lifecycle/persistence.md) for the full write API and the
difference between lifecycle-aware and lifecycle-bypassing methods.

## Validations and Callbacks

Validations decide whether a model can be saved:

```crystal
class Book
  include Lustra::Model

  primary_key
  column title : String

  def validate
    if title_column.defined? && title.empty?
      add_error("title", "must not be blank")
    end
  end
end
```

Callbacks run application code around lifecycle events:

```crystal
class Book
  include Lustra::Model

  column title : String

  after(:create) do |model|
    book = model.as(Book)
    puts "Created #{book.title}"
  end
end
```

See [Validations](lifecycle/validations.md) and
[Triggers](lifecycle/callbacks.md).

## Migrations

Migrations describe database schema changes:

```crystal
create_table(:books) do |t|
  t.column :title, :string, null: false
  t.column :author_name, :string, null: false
  t.timestamps
end
```

See [Manage migrations](../migrations/manage-migrations.md).

## Associations

Associations describe relationships between models:

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

  belongs_to author : Author
end
```

`Book` stores `author_id`, while `Author#books` returns a query collection of
matching books.

See [Associations](associations/README.md).

## Querying

Model queries start with `Model.query`:

```crystal
Book.query
  .where(author_name: "J. R. R. Tolkien")
  .order_by(title: :asc)
  .limit(10)
```

Query methods refine the SQL. Terminal methods such as `each`, `to_a`, `first`,
`count`, `empty?`, and `exists?` execute it.

See [Querying with Lustra](../querying/README.md).
