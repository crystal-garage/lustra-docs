# Insert Clause

Start an insert query with `Lustra::SQL.insert_into` or the shorter `Lustra::SQL.insert` alias.

```crystal
Lustra::SQL.insert_into(:users, {email: "admin@example.com", active: true}).execute

Lustra::SQL.insert(:users, {email: "admin@example.com"}).execute
```

## Values

You can pass values when building the query, or call `values` later.

```crystal
query = Lustra::SQL.insert_into(:users)
  .values({email: "admin@example.com", active: true})

query.to_sql
# => INSERT INTO "users" ("email", "active") VALUES ('admin@example.com', TRUE)
```

Multiple rows can be inserted by passing multiple named tuples or an array.

```crystal
Lustra::SQL.insert_into(
  :users,
  {email: "one@example.com"},
  {email: "two@example.com"}
).execute

Lustra::SQL.insert_into(:users).values([
  {email: "one@example.com"},
  {email: "two@example.com"},
]).execute
```

When no values are provided, Lustra renders `DEFAULT VALUES`.

```crystal
Lustra::SQL.insert_into(:users).to_sql
# => INSERT INTO "users" DEFAULT VALUES
```

## Insert From Select

Use a select builder as the insert source.

```crystal
Lustra::SQL.insert_into(:users)
  .values(
    Lustra::SQL.select.from(:old_users).where { old_users.id > 100 }
  )
  .execute
```

## Returning

Use `returning` to ask PostgreSQL to return inserted columns.

```crystal
row = Lustra::SQL.insert_into(:users, {email: "admin@example.com"})
  .returning("*")
  .execute

id = row["id"]
```

Without `returning`, `execute` runs the query and returns an empty hash.

## Conflict Handling

Use `on_conflict` with `do_nothing` or `do_update`.

```crystal
Lustra::SQL.insert_into(:users, {email: "admin@example.com"})
  .on_conflict("(email)")
  .do_nothing
  .execute
```

```crystal
Lustra::SQL.insert_into(:users, {email: "admin@example.com", visits: 1})
  .on_conflict("(email)")
  .do_update do |update|
    update.set(visits: Lustra::SQL.unsafe("users.visits + 1"))
  end
  .execute
```

`on_conflict` also accepts an expression block for partial-index style conditions.

```crystal
Lustra::SQL.insert_into(:users, {email: "admin@example.com"})
  .on_conflict { active == true }
  .do_nothing
```

## Raw Values

Use `Lustra::SQL.unsafe` only for trusted SQL expressions.

```crystal
Lustra::SQL.insert_into(:events, {created_at: Lustra::SQL.unsafe("NOW()")}).execute
```
