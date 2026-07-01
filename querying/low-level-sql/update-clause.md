# Update Clause

Start an update query with `Lustra::SQL.update`.

```crystal
Lustra::SQL.update(:users)
  .set(active: false)
  .where(id: 10)
  .execute
```

Use `execute_and_count` when you need the number of affected rows.

```crystal
affected = Lustra::SQL.update(:users)
  .set(active: false)
  .where { last_seen_at < Time.utc(2024, 1, 1) }
  .execute_and_count
```

## Setting Values

`set` accepts keyword arguments, a named tuple, a string-keyed hash, or a raw SQL fragment.

```crystal
Lustra::SQL.update(:users).set(active: true, name: "Jane")

Lustra::SQL.update(:users).set({active: true, name: "Jane"})

fields = {} of String => Lustra::SQL::Any
fields["active"] = true
Lustra::SQL.update(:users).set(fields)
```

Raw `SET` fragments are inserted as SQL. Use them only for trusted SQL.

```crystal
Lustra::SQL.update(:counters)
  .set(%("value" = "value" + 1))
  .where(id: 1)
```

For raw expressions assigned to a column, use `Lustra::SQL.unsafe`.

```crystal
Lustra::SQL.update(:events)
  .set(updated_at: Lustra::SQL.unsafe("NOW()"))
  .where(id: 1)
```

## Filtering

Update queries include the same `where` helpers as select queries.

```crystal
Lustra::SQL.update(:users)
  .set(active: false)
  .where("email LIKE ?", "%@example.com")

Lustra::SQL.update(:users)
  .set(active: false)
  .where { users.role == "guest" }
```

## From a Collection or Select Query

Model collections and select builders can be converted to update queries with `to_update`. The new query reuses the source table and `WHERE` clauses.

```crystal
User.query
  .where(active: false)
  .to_update
  .set(archived: true)
  .execute_and_count
```
