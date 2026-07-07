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

Pass a block to refine the generated `Lustra::SQL::InsertQuery`.

```crystal
User.import(users) do |query|
  query.on_conflict("(id)").do_update do |update|
    update.set(first_name: Lustra::SQL.unsafe("EXCLUDED.first_name"))
  end
end
```

The block can use the same conflict helpers as low-level inserts: `on_conflict`, `do_nothing`, and `do_update`.

```crystal
User.import(users) do |query|
  query.on_conflict("(id)").do_nothing
end
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

`insert` returns a raw result hash when PostgreSQL returns a row, or `nil` when no row is returned.

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

Every row passed to `insert_all` must have the same keys. `insert_all` returns raw result hashes from PostgreSQL.

Control the `RETURNING` clause with `returning`.

```crystal
Book.insert_all(rows, returning: [:id, :title])
Book.insert_all(rows, returning: "id, title AS book_title")
Book.insert_all(rows, returning: false)
```

By default, `insert_all` skips rows that conflict with any unique index PostgreSQL reports through `ON CONFLICT DO NOTHING`.

Pass `unique_by` when duplicates should be checked against one specific unique constraint or unique index.

```crystal
Book.insert_all(rows, unique_by: :isbn, returning: false)
```

`unique_by` can be one column or several columns. The database must have a matching unique constraint or unique index.

`record_timestamps` is reserved for API compatibility but is not supported yet. Passing a truthy value raises an error.

## Upserts

Use `Model.upsert` to insert one row or update the existing row when a conflict happens.

```crystal
book = Book.upsert({
  title: "Rework",
  author: "David Heinemeier Hansson",
  isbn: "9780307463746",
}, unique_by: :isbn)
```

Use `Model.upsert_all` to apply the same behavior to many rows with one SQL statement.

```crystal
books = Book.upsert_all([
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

`upsert` returns a model instance, or `nil` when `on_duplicate: :skip` skips the row. `upsert_all` returns model instances for rows returned by PostgreSQL.

Upsert methods bypass validations and callbacks. They use PostgreSQL `ON CONFLICT`, so they need a unique constraint or unique index for the conflict target. They always use `RETURNING *` and return model instances built from the rows PostgreSQL returns.

If a single `upsert_all` batch contains two rows that conflict with the same existing row, PostgreSQL cannot update that target row twice in one statement. Deduplicate incoming data before calling `upsert_all` when duplicates are possible inside the same batch.

## Which Insert Method To Use

| Method | Use when | Validations / callbacks | Return value |
| :--- | :--- | :--- | :--- |
| `import` | You already have new model instances and need model lifecycle behavior. | Runs validations and save/create callbacks. | Persisted model instances. |
| `insert` | You need one direct insert. | Skipped. | Raw result hash or `nil`. |
| `insert_all` | You need one SQL statement for many direct inserts. | Skipped. | Raw result hashes. |
| `upsert` | You need one insert-or-update by a unique key. | Skipped. | Model instance or `nil`. |
| `upsert_all` | You need many insert-or-update rows in one SQL statement. | Skipped. | Model instances returned by PostgreSQL. |

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
