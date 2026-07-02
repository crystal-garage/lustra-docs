# Polymorphic Models

Use `polymorphic` when several Crystal model classes share one PostgreSQL
table and are distinguished by a type column.

If you are choosing between polymorphic models and polymorphic associations, see
[Polymorphic Models vs Associations](polymorphic-models-vs-associations.md).

## Migration

The shared table needs a type column. The default type column name is `type`,
but you can choose another name with `polymorphic through: "column_name"`.

```crystal
create_table(:events) do |t|
  t.column :type, :string, null: false, index: true
  t.column :message, :string, null: false
  t.column :user_id, :bigint
  t.column :repository_id, :bigint
end
```

## Model Setup

Declare `polymorphic` on the base model. Subclasses inherit the same table and
store their class name in the type column.

```crystal
abstract class Event
  include Lustra::Model

  self.table = "events"

  polymorphic through: "type"

  primary_key
  column message : String
end

class UserEvent < Event
  column user_id : Int64
end

class RepositoryEvent < Event
  column repository_id : Int64
end
```

When a subclass is validated and saved, Lustra assigns the type column from the
model class name.

```crystal
event = UserEvent.create!(
  message: "User signed in",
  user_id: user.id
)

event.type # => "UserEvent"
```

## Querying

Querying the base class returns model instances built as the matching subclass:

```crystal
Event.query.each do |event|
  case event
  when UserEvent
    puts event.user_id
  when RepositoryEvent
    puts event.repository_id
  end
end
```

Querying a subclass automatically filters by the type column:

```crystal
UserEvent.query.count
RepositoryEvent.query.count
```

The base query sees all rows in the shared table:

```crystal
Event.query.count
```

## Callbacks and Validations

Callbacks and validations declared on the base model and the subclass both run
for subclass records.

```crystal
abstract class Event
  include Lustra::Model

  polymorphic through: "type"

  before(:validate) do |model|
    # runs for UserEvent and RepositoryEvent
  end
end

class UserEvent < Event
  before(:validate) do |model|
    # runs only for UserEvent
  end
end
```
