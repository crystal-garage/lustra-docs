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

The next page covers column options in more detail.
