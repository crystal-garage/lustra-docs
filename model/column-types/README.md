# Defining Your Model

Model definition in Lustra starts by including `Lustra::Model` in a class and declaring the columns that exist in the PostgreSQL table.

Assume this table:

```sql
CREATE TABLE articles (
  id serial PRIMARY KEY,
  name text NOT NULL,
  description text
);
```

The matching model can be written as:

{% code title="article.cr" %}
```crystal
class Article
  include Lustra::Model

  column id : Int32, primary: true, presence: false
  column name : String
  column description : String?
end
```
{% endcode %}

Step by step:

```crystal
include Lustra::Model
```

This adds Lustra's model behavior to the class.

```crystal
column name : String
```

This maps the PostgreSQL `name` column to a Crystal `String` attribute.

```crystal
column description : String?
```

This maps the nullable PostgreSQL `description` column to a nilable Crystal attribute.

```crystal
column id : Int32, primary: true, presence: false
```

This declares `id` as the primary key. Because PostgreSQL generates the `serial` value, `presence: false` tells Lustra not to require the value before insert.

You can then use the model:

```crystal
article = Article.new({name: "A superb article!"})
article.description = "This is a masterpiece."
article.save!

puts "Article saved as id=#{article.id}"
```

By default, Lustra infers the table name from the model name. `Article` maps to `articles`.

You can override the table name:

```crystal
class Model::Customer
  include Lustra::Model

  self.table = "clients"

  column id : Int64, primary: true, presence: false
  column name : String
end
```

## PostgreSQL Schemas

Set `self.schema` to qualify a model's table name:

```crystal
class ArchivedArticle
  include Lustra::Model

  self.schema = "archive"
  self.table = "articles"

  column id : Int32, primary: true, presence: false
  column name : String
end

ArchivedArticle.query.to_sql
# => SELECT * FROM "archive"."articles"
```

The schema and table must already exist. Without an explicit schema, Lustra uses an unqualified table name, which PostgreSQL resolves through its search path. Setting `self.schema` is separate from choosing a database connection with `self.connection`.

## Inspecting the Database Schema

`Model.schema_description` queries PostgreSQL for the model's table metadata using the model's configured connection:

```crystal
info = ArchivedArticle.schema_description

info.schema # => "archive"
info.table  # => "articles"

info.columns.each do |column|
  puts "#{column.name}: #{column.data_type}, nullable: #{column.nullable}"
end

info.indexes.each do |index|
  puts "#{index.name}: #{index.definition}"
end
```

The returned `SchemaDescription` contains `schema`, `table`, `columns`, and `indexes`. Column metadata exposes `name`, `data_type`, `nullable`, `default_value`, `collation`, and `description`. Index metadata exposes `name`, `definition`, `unique`, and `primary`.

Introspection uses the configured schema, or `public` when none is set. It does not generate model columns; declare those with `column` as usual.

The next page covers column options in more detail.
