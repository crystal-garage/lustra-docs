# Locks

Lustra exposes PostgreSQL locks in two ways:

- row locks with `with_lock` on select and model collection queries
- table locks with `Lustra::SQL.lock`

Use locks inside a transaction so the lock is held for the intended unit of work.

## Row Locks

`with_lock` appends a lock clause to the generated `SELECT`.

```crystal
Lustra::SQL.transaction do
  User.query
    .where(organization: "Crystal Lang")
    .with_lock
    .each do |user|
      user.update!(active: false)
    end
end
```

By default, `with_lock` uses `FOR UPDATE`.

```sql
SELECT * FROM "users" WHERE ("organization" = 'Crystal Lang') FOR UPDATE
```

Pass a custom PostgreSQL lock clause when needed.

```crystal
User.query
  .where(processed: false)
  .with_lock("FOR UPDATE SKIP LOCKED")
  .limit(100)
  .each do |job|
    job.update!(processed: true)
  end
```

The same method is available on low-level select builders.

```crystal
Lustra::SQL.select
  .from(:jobs)
  .where(processed: false)
  .with_lock("FOR UPDATE SKIP LOCKED")
  .to_a
```

## Table Locks

Use `Lustra::SQL.lock` to lock a full table for the duration of a block.

```crystal
Lustra::SQL.lock(:repositories) do
  Lustra::SQL.insert(:repositories, {name: "lustra"}).execute
end
```

The default lock mode is `ACCESS EXCLUSIVE`.

Pass another PostgreSQL lock mode as the second argument.

```crystal
Lustra::SQL.lock(:repositories, "SHARE") do
  count = Repository.query.count
end
```

`Lustra::SQL.lock` opens a transaction internally and yields the checked-out connection.

```crystal
Lustra::SQL.lock(:repositories) do |connection|
  connection.exec("SELECT 1")
end
```

PostgreSQL supports lock modes such as `ACCESS SHARE`, `ROW SHARE`, `ROW EXCLUSIVE`, `SHARE UPDATE EXCLUSIVE`, `SHARE`, `SHARE ROW EXCLUSIVE`, `EXCLUSIVE`, and `ACCESS EXCLUSIVE`.

See the PostgreSQL locking documentation for the exact compatibility rules between lock modes.
