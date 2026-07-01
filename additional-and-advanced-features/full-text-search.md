# Full Text Search

Lustra integrates with PostgreSQL `tsvector` search through migration helpers and a model macro.

## Migration

Use `t.full_text_searchable` inside `create_table`.

```crystal
create_table(:posts) do |t|
  t.column :title, :string, null: false
  t.column :content, :string, null: false

  t.full_text_searchable on: [{"title", 'A'}, {"content", 'C'}]
end
```

This creates:

- a `tsvector` column
- a GIN index
- a trigger
- a trigger function that updates the vector column

The default vector column is `full_text_vector`.

For an existing table, use `add_full_text_searchable`.

```crystal
add_full_text_searchable(
  "posts",
  [{"title", 'A'}, {"content", 'C'}]
)
```

## Model

Add `full_text_searchable` to the model.

```crystal
class Post
  include Lustra::Model

  column title : String
  column content : String

  full_text_searchable
end
```

This defines a `search` scope by default.

```crystal
Post.query.search("orm")
```

Search is chainable.

```crystal
user = User.query.find_by! { email == "some_email@example.com" }
Post.query.from_user(user).search("orm")
```

## Custom Vector Column

Pass the vector column name in the migration and in the model.

```crystal
create_table(:series) do |t|
  t.column :title, :string
  t.column :description, :string
  t.full_text_searchable on: [{"title", 'A'}, {"description", 'C'}], column_name: "tsv"
end
```

```crystal
class Series
  include Lustra::Model

  full_text_searchable "tsv"
end
```

## Catalog

The default catalog is `pg_catalog.english`.

```crystal
t.full_text_searchable(
  on: [{"title", 'A'}, {"content", 'C'}],
  catalog: "pg_catalog.french"
)
```

Use the same catalog in the model macro.

```crystal
full_text_searchable catalog: "pg_catalog.french"
```

## Custom Scope Name

The model macro accepts a custom scope name.

```crystal
full_text_searchable "full_text_vector", scope_name: "text_search"

Post.query.text_search("crystal")
```

## Query Syntax

`search` converts end-user text into a PostgreSQL `to_tsquery` string.

```crystal
Post.query.search("breaking")
Post.query.search("break -prison")
Post.query.search("\"exact phrase\"")
```

Internally, Lustra uses `Lustra::Model::FullTextSearchable.to_tsq` to escape and join terms.
