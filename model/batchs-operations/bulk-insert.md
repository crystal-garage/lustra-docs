# Bulk Insert And Delete

## Bulk Insert

Use `Model.import` to insert multiple model instances with one `INSERT` query.

```crystal
users = [
  User.new({id: 1, first_name: "x"}),
  User.new({id: 2, first_name: "y"}),
  User.new({id: 3, first_name: "z"}),
]

saved_users = User.import(users)
```

`import` returns new persisted model instances loaded from PostgreSQL with `RETURNING *`. The original model instances are not changed.

Every imported model must be new. Lustra raises if any item is already persisted.

```crystal
persisted = User.create!({first_name: "already saved"})
User.import([persisted])
# raises
```

`import` validates each model before inserting. It also runs `:save` and `:create` callbacks around the import.

## Conflict Handling

Pass an `on_conflict` Proc to refine the generated `Lustra::SQL::InsertQuery`.

```crystal
User.import(users, on_conflict: ->(query : Lustra::SQL::InsertQuery) do
  query.on_conflict("(id)").do_update do |update|
    update.set(first_name: Lustra::SQL.unsafe("EXCLUDED.first_name"))
  end
end)
```

The Proc can use the same conflict helpers as low-level inserts: `on_conflict`, `do_nothing`, and `do_update`.

```crystal
User.import(users, on_conflict: ->(query : Lustra::SQL::InsertQuery) do
  query.on_conflict("(id)").do_nothing
end)
```

## Direct Inserts

Use `Model.insert` to insert one row without building a model object, running validations, or running callbacks.

```crystal
Book.insert({
  title: "Rework",
  author: "David Heinemeier Hansson",
  isbn: "9780307463746",
})
```

`insert` returns an `Int64` affected-row count: `1` for an inserted row or `0` when a conflict skips it.

Use `Model.insert_all` to insert many rows with one SQL statement.

```crystal
Book.insert_all([
  {
    title: "Rework",
    author: "David Heinemeier Hansson",
    isbn: "9780307463746",
  },
  {
    title: "Eloquent Ruby",
    author: "Russ Olsen",
    isbn: "9780321584106",
  },
])
```

Every row passed to `insert_all` must have the same keys. `insert_all` returns the number of inserted rows as `Int64`.

Pass a named tuple of column names and Crystal types to `returning`:

```crystal
Book.insert(row, returning: {id: Int64, title: String})
# => Tuple(Int64, String)?

Book.insert_all(rows, returning: {id: Int64, title: String})
# => Array(Tuple(Int64, String))
```

The types must match the database columns. Tuple values follow the declared column order. PostgreSQL does not guarantee the order of returned rows. Skipped rows are omitted; a skipped single-row insert returns `nil` with `returning`. Omit `returning` to get an affected-row count. Arrays, SQL strings, and booleans are not accepted by these model-level `returning` overloads.

By default, `insert_all` skips rows that conflict with any unique index PostgreSQL reports through `ON CONFLICT DO NOTHING`.

Pass `unique_by` when duplicates should be checked against one specific unique constraint or unique index.

```crystal
Book.insert_all(rows, unique_by: :isbn)
```

`unique_by` can be one column or several columns. The database must have a matching unique constraint or unique index.

`record_timestamps` is reserved for API compatibility but is not supported yet. Passing a truthy value raises an error.

## Upserts

Use `Model.upsert` to insert one row or update the existing row when a conflict happens.

```crystal
affected = Book.upsert({
  title: "Rework",
  author: "David Heinemeier Hansson",
  isbn: "9780307463746",
}, unique_by: :isbn)
```

Use `Model.upsert_all` to apply the same behavior to many rows with one SQL statement.

```crystal
affected = Book.upsert_all([
  {
    title: "Rework",
    author: "David Heinemeier Hansson",
    isbn: "9780307463746",
  },
  {
    title: "Eloquent Ruby",
    author: "Russ Olsen",
    isbn: "9780321584106",
  },
], unique_by: :isbn)
```

By default, upsert uses `on_duplicate: :update`. Pass `on_duplicate: :skip` to keep existing rows unchanged.

```crystal
Book.upsert_all(rows, unique_by: :isbn, on_duplicate: :skip)
```

Limit which columns are updated with `update_only`.

```crystal
Book.upsert_all(rows, unique_by: :isbn, update_only: [:title, :author])
```

For PostgreSQL expressions beyond replacing columns with `excluded` values, pass a custom `SET` clause with `Lustra::SQL.unsafe`.

```crystal
Book.upsert(
  {
    isbn: "9780307463746",
    inventory_count: 3,
  },
  unique_by: :isbn,
  on_duplicate: Lustra::SQL.unsafe(
    %("inventory_count" = "books"."inventory_count" + excluded."inventory_count")
  )
)
```

A custom conflict update cannot be combined with `update_only`. Because Lustra inserts the unsafe fragment directly into the SQL statement, never build it from untrusted input.

By default, `upsert` and `upsert_all` return an `Int64` affected-row count. They do not return model instances.

```crystal
Book.upsert(row, unique_by: :isbn)      # Int64
Book.upsert_all(rows, unique_by: :isbn) # Int64

Book.upsert(row, unique_by: :isbn, returning: {id: Int64, title: String})
# => Tuple(Int64, String)?

Book.upsert_all(rows, unique_by: :isbn, returning: {id: Int64, title: String})
# => Array(Tuple(Int64, String))
```

When `on_duplicate: :skip` skips a row, it contributes zero to the count. With typed `returning`, a skipped single-row upsert returns `nil`, and skipped rows are omitted from bulk results.

Upsert methods bypass validations and callbacks. They use PostgreSQL `ON CONFLICT`, so they need a unique constraint or unique index for the conflict target.

If a single `upsert_all` batch contains two rows that conflict with the same existing row, PostgreSQL cannot update that target row twice in one statement. Deduplicate incoming data before calling `upsert_all` when duplicates are possible inside the same batch.

## Which Insert Method To Use

| Method | Use when | Validations / callbacks | Return value |
| :--- | :--- | :--- | :--- |
| `import` | You already have new model instances and need model lifecycle behavior. | Runs validations and save/create callbacks. | Persisted model instances. |
| `insert` | You need one direct insert. | Skipped. | `Int64` count; typed tuple or `nil` with `returning`. |
| `insert_all` | You need one SQL statement for many direct inserts. | Skipped. | `Int64` count; array of typed tuples with `returning`. |
| `upsert` | You need one insert-or-update by a unique key. | Skipped. | `Int64` count; typed tuple or `nil` with `returning`. |
| `upsert_all` | You need many insert-or-update rows in one SQL statement. | Skipped. | `Int64` count; array of typed tuples with `returning`. |

## Low-Level Bulk Insert

When you do not need model validation or callbacks, use the low-level insert builder.

```crystal
Lustra::SQL.insert_into(:users, [
  {first_name: "x"},
  {first_name: "y"},
  {first_name: "z"},
]).execute
```

This path sends SQL directly and does not instantiate models.

## Bulk Delete

Use `delete_all` to delete rows matching a collection without loading models.

```crystal
User.query.where(active: false).delete_all
```

`delete_all` bypasses model callbacks.

Use `destroy_all` when you need destroy callbacks for each record.

```crystal
User.query.where(active: false).destroy_all
```

`destroy_all` loads each model and calls `destroy`, so it is slower but preserves lifecycle behavior.
