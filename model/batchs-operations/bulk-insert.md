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
