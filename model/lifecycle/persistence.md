# Persistence

## Save

`save` inserts or updates a model:

```crystal
user = User.new({email: "user@example.com"})

if user.save
  puts user.id
else
  puts user.print_errors
end
```

`save` returns `true` when the record was persisted. It returns `false` when validations fail or conflict handling skips an insert.

`save!` raises `Lustra::Model::InvalidError` when validations fail or conflict handling skips an insert. It returns the model on success:

```crystal
user = User.new({email: "user@example.com"})
user.save!
```

When the model is new, Lustra runs an `INSERT`. When the model is already persisted and has changed columns, Lustra runs an `UPDATE`.

## Conflict-Skipped Inserts

An insert skipped by `ON CONFLICT DO NOTHING` leaves the model unpersisted. Its attributes and existing dirty state are preserved, and successful-create and successful-save callbacks do not run.

```crystal
# Assumes a unique email constraint and an existing row with this address.
user = User.new({email: "existing@example.com"})
saved = user.save(->(query : Lustra::SQL::InsertQuery) do
  query.on_conflict("(email)").do_nothing
end)

saved           # => false
user.persisted? # => false
```

`save_with_associations` also returns `false` when the main model insert is skipped. `save!` raises `Lustra::Model::InvalidError` for this outcome.

## Update

`update` assigns attributes and calls `save`:

```crystal
user.update(email: "new@example.com")
```

`update!` assigns attributes and calls `save!`:

```crystal
user.update!(email: "new@example.com")
```

## Idempotent Sync

Use `find_or_build` when synchronizing records from an external source. It
returns the existing record when one matches the lookup fields, or builds a new
unsaved model with those fields assigned.

```crystal
user = User.query.find_or_build(provider: "github", provider_id: github_user.id)

user.login = github_user.login
user.name = github_user.name
user.avatar_url = github_user.avatar_url
user.synced_at = Time.utc
user.save!
```

This lets the sync code assign the latest external values and save once.

Use `find_or_create` when the initial values are enough to create the record
immediately:

```crystal
tag = Tag.query.find_or_create(name: "orm")
```

`find_or_create` is also useful for join records:

```crystal
RepositoryFork.query.find_or_create(
  parent_id: parent_repository.id,
  fork_id: repository.id
)
```

Prefer `find_or_build` when the record needs more fields assigned before saving.
Prefer `find_or_create` when the lookup fields are enough for a valid record.

## Checking for Pending Persistence

`modified?` returns `true` for a new record or a persisted record with changed attributes. `changed?` checks dirty attributes only; a new model can need an insert even when `changed?` is false.

```crystal
user = User.new({email: "user@example.com"})
user.modified? # => true

user.save!
user.modified? # => false

user.email = "new@example.com"
user.modified? # => true
```

## Direct Updates and Dirty Tracking

`update_column`, `update_columns`, `increment!`, and `decrement!` clear dirty state only for the attributes they write. Unrelated pending edits remain available for a later save.

```crystal
user = User.find!(1)
user.email = "pending@example.com"
user.update_column(:active, false)

user.email_column.changed? # => true
user.save! # persists the pending email change
```

`increment!` and `decrement!` use `UPDATE ... RETURNING`: they apply the change to the database value and update the local counter in one statement. They return the model, bypass validations and callbacks, and raise `Lustra::SQL::RecordNotFoundError` if the row no longer exists.

The non-bang `increment` and `decrement` methods only change the current in-memory value; call `save!` to persist it.

## Reload

`reload` fetches the current database row by primary key, replaces the model values, clears cached association data, and marks the model as persisted:

```crystal
user.reload
```

## Delete and Destroy

`delete` removes the row directly and does not run callbacks:

```crystal
user.delete
```

`destroy` removes the row and runs `:destroy` callbacks:

```crystal
user.destroy
```

Use `destroy` when application cleanup belongs in callbacks. Use `delete` only when you intentionally want a direct delete.

## Import

`Model.import` inserts many new models in one SQL query:

```crystal
users = [
  User.new({email: "one@example.com"}),
  User.new({email: "two@example.com"}),
]

saved_users = User.import(users)
```

Each imported model must be new. Lustra validates each model and runs save/create callbacks around the import.

## Read-Only Models

Set `self.read_only = true` for models that should not be saved, such as database views or system catalog models:

```crystal
class RepositoryStats
  include Lustra::Model

  self.table = "repository_stats"
  self.read_only = true

  column repository_id : Int64, primary: true
  column forks_count : Int64
end
```

`save` returns `false` for read-only models. `save!` raises `Lustra::Model::ReadOnlyError` with a message explaining that the model is read-only.

## Lifecycle-Bypassing Methods

Some methods intentionally update the database directly. They do not run validations, callbacks, or automatic timestamp updates unless stated otherwise.

| Method | Behavior |
| :--- | :--- |
| `delete` | Deletes one persisted model without callbacks. |
| `Model.insert` | Inserts one row directly without building a model. |
| `Model.insert_all` | Inserts many rows directly in one SQL statement. |
| `Model.upsert` | Inserts or updates one row through PostgreSQL `ON CONFLICT`. |
| `Model.upsert_all` | Inserts or updates many rows through PostgreSQL `ON CONFLICT`. |
| `update_column` | Updates one column directly. |
| `update_columns` | Updates multiple columns directly. |
| `increment!` / `decrement!` | Atomically updates a numeric column in the database. |
| `touch` | Updates timestamp columns through `update_columns`; it bypasses validations and callbacks. |
| `Collection#delete_all` | Bulk deletes matching rows without loading models. |
| `Collection#update_all` | Bulk updates matching rows without loading models. |

Use lifecycle-bypassing methods for counters, flags, maintenance tasks, and bulk operations where skipping callbacks is intentional.

Use `destroy`, `save`, `save!`, `update`, or `update!` when validations and callbacks matter.
