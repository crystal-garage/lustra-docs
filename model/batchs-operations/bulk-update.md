# Bulk Update

Use `update_all` to update every row matched by a collection without loading models.

```crystal
affected = User.query
  .where(active: false)
  .update_all(active: true)

puts "Updated #{affected} users"
```

`update_all` returns the number of affected rows.

```crystal
affected = User.query
  .where { id.in?([1, 2]) }
  .update_all(first_name: "Updated", last_name: "User")
```

You can pass a named tuple or a string-keyed hash.

```crystal
User.query.where(active: false).update_all({active: true})

fields = {} of String => Lustra::SQL::Any
fields["active"] = true
User.query.where(active: false).update_all(fields)
```

`update_all` bypasses validations, callbacks, change tracking, and automatic timestamp updates. Use it only when direct SQL behavior is what you want.

## Custom Update Queries

Use `to_update` when you need lower-level update builder features.

```crystal
User.query
  .where(active: false)
  .to_update
  .set(last_seen_at: Lustra::SQL.unsafe("NOW()"))
  .execute_and_count
```

`to_update` copies the collection's table and `WHERE` clauses into a `Lustra::SQL::UpdateQuery`.

## Bulk Delete

Use `delete_all` to delete every row matched by a collection without loading models.

```crystal
User.query.where(active: false).delete_all
```

`delete_all` bypasses destroy callbacks.

Use `destroy_all` when callbacks must run.

```crystal
User.query.where(active: false).destroy_all
```

## Custom Delete Queries

Use `to_delete` when you need the lower-level delete builder.

```crystal
User.query
  .where(active: false)
  .to_delete
  .execute_and_count
```

`to_delete` copies the collection's table and `WHERE` clauses into a `Lustra::SQL::DeleteQuery`.

## Limitations

`to_update` and `to_delete` require a query with exactly one source table. They are not meant for collections built from multiple tables or subqueries.
