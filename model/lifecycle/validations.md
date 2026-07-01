# Validations

Validations decide whether a model can be saved.

`valid?` clears existing errors, runs validation callbacks, calls `validate`, checks column presence rules, and returns `true` or `false`:

```crystal
user = User.new

unless user.valid?
  puts user.print_errors
end
```

`valid!` raises `Lustra::Model::InvalidError` when the model is invalid.

## Presence Validation

Lustra derives basic presence checks from column types and options:

```crystal
column name : String
column nickname : String?
column id : Int64, primary: true, presence: false
```

Rules:

| Column | Behavior |
| :--- | :--- |
| `column name : String` | Must be present before save. |
| `column nickname : String?` | May be nil. |
| `presence: false` | Skips the presence check, useful for database-generated values such as `serial` primary keys. |

Database `NOT NULL` constraints are still recommended. Model validations are application-level checks, not a replacement for database constraints.

## Custom Validation

Override `validate` to add custom rules:

```crystal
class Article
  include Lustra::Model

  column id : Int64, primary: true, presence: false
  column title : String
  column body : String

  def validate
    if body_column.defined? && body.size < 100
      add_error("body", "must contain at least 100 characters")
    end
  end
end
```

Check `xxx_column.defined?` before reading a column that may not have been fetched or assigned. Reading an uninitialized column raises an error.

## Validation Helpers

`on_presence` runs a block only when all listed columns are defined:

```crystal
def validate
  on_presence(:body) do
    add_error("body", "must contain at least 100 characters") if body.size < 100
  end
end
```

`ensure_than` adds an error when the block returns false:

```crystal
def validate
  ensure_than :body, "must contain at least 100 characters" do |value|
    value.size >= 100
  end
end
```

## Errors

Validation errors are stored in `errors`:

```crystal
article = Article.new

unless article.valid?
  article.errors.each do |error|
    puts "#{error.column}: #{error.reason}"
  end
end
```

Use `add_error(reason)` for model-level errors and `add_error(column, reason)` for column-specific errors.

`print_errors` returns a grouped string representation that is useful for logs and exception messages.
