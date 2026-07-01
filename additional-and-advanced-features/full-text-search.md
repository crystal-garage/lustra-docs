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

## Manual Vector Maintenance

`t.full_text_searchable` is a good fit when the search vector is built only from
columns on the same table. It creates and maintains the vector for those
same-table fields.

If the vector depends on joined or aggregated data, create the `tsvector`
column, index, and PostgreSQL triggers manually, then use the model macro only
for querying.

For example, a repository search vector may include repository fields and tag
names from a join table:

```crystal
create_table(:repositories) do |t|
  t.column :name, :string, null: false
  t.column :description, :string
  t.column :tsv, "tsvector"

  t.index :tsv, using: :gin
end
```

Then create PostgreSQL trigger functions in a migration:

```crystal
execute <<-SQL
  -- Keep tsv in sync when inserting a new repository.
  -- It computes a weighted vector from repository name and description.
  CREATE OR REPLACE FUNCTION tsv_trigger_insert_repositories() RETURNS trigger AS $$
  begin
    new.tsv :=
      setweight(to_tsvector('pg_catalog.simple', coalesce(new.name, '')), 'A') ||
      setweight(to_tsvector('pg_catalog.simple', coalesce(new.description, '')), 'B');
    return new;
  end
  $$ LANGUAGE plpgsql;
  SQL

execute <<-SQL
  -- Rebuild tsv on UPDATE of repository rows.
  -- It uses current repository fields plus all joined tag names,
  -- so changes in title/body or related tags can affect search ranking.
  CREATE OR REPLACE FUNCTION tsv_trigger_update_repositories() RETURNS trigger AS $$
  begin
    SELECT setweight(to_tsvector('pg_catalog.simple', coalesce(r.name, '')), 'A') ||
           setweight(to_tsvector('pg_catalog.simple', coalesce(r.description, '')), 'B') ||
           setweight(to_tsvector('pg_catalog.simple', coalesce((string_agg(tags.name, ' ')), '')), 'C')
      INTO new.tsv
      FROM repositories r
      LEFT JOIN repository_tags ON repository_tags.repository_id = r.id
      LEFT JOIN tags ON tags.id = repository_tags.tag_id
      WHERE r.id = new.id
      GROUP BY r.id;
    return new;
  end
  $$ LANGUAGE plpgsql;
  SQL
```

Attach the functions to the table:

```crystal
execute <<-SQL
  -- Fire on INSERT so a new row always has an initial tsvector value.
  CREATE TRIGGER tsv_insert_repositories BEFORE INSERT
    ON repositories
    FOR EACH ROW
    EXECUTE PROCEDURE tsv_trigger_insert_repositories();
  SQL

execute <<-SQL
  -- Fire on UPDATE so full-text content reflects changed repository data.
  CREATE TRIGGER tsv_update_repositories BEFORE UPDATE
    ON repositories
    FOR EACH ROW
    EXECUTE PROCEDURE tsv_trigger_update_repositories();
  SQL
```

Finally, point the model macro at the manually maintained vector column:

```crystal
class Repository
  include Lustra::Model

  column name : String
  column description : String?

  full_text_searchable "tsv", catalog: "pg_catalog.simple"
end
```

```crystal
Repository.query.search("orm")
```

The migration owns how `tsv` is maintained. Lustra only builds the search
condition against that column.

When the vector includes joined data, make sure the vector is refreshed when
that joined data changes. The example above refreshes `tsv` when a repository is
inserted or updated. If tag changes must update search results immediately, add
triggers on the join table or update the repository row from application code so
the repository trigger recalculates the vector.

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
