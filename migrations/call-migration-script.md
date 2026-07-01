# Call Migrations From Code

You can run migrations directly through `Lustra::Migration::Manager`. This is useful for custom CLIs, test setup, or applications that manage database setup inside Crystal code.

Before calling the manager, initialize the database connection and require your migration files.

```crystal
require "lustra"
require "./db/migrations/*"

Lustra::SQL.init(ENV["DATABASE_URL"])

Lustra::Migration::Manager.instance.apply_all
```

## Apply Pending Migrations

`apply_all` applies every registered migration that is not already recorded in `__lustra_metadatas`.

```crystal
Lustra::Migration::Manager.instance.apply_all
```

The manager sorts migrations by `uid` before applying them.

## Move To A Version

Use `apply_to` to move the schema to a specific migration id. Lustra applies missing migrations below that id and rolls back applied migrations above it.

```crystal
Lustra::Migration::Manager.instance.apply_to(202607010001_i64)
```

You can restrict the direction.

```crystal
Lustra::Migration::Manager.instance.apply_to(202607010001_i64, direction: :up)
Lustra::Migration::Manager.instance.apply_to(202607010001_i64, direction: :down)
Lustra::Migration::Manager.instance.apply_to(202607010001_i64, direction: :both)
```

## Force One Migration Up Or Down

Use `up` or `down` when you want to run one specific migration by id.

```crystal
manager = Lustra::Migration::Manager.instance

manager.up(202607010001_i64)
manager.down(202607010001_i64)
```

`up` raises if the migration is already applied. `down` raises if the migration is not applied.

## Status

Use `print_status` to inspect known migrations.

```crystal
puts Lustra::Migration::Manager.instance.print_status
```

The output marks each registered migration as applied or pending.

## Refreshing State

The manager keeps migration state in memory after it loads metadata from the database. Use `refresh` to reload the applied migration ids from `__lustra_metadatas`.

```crystal
Lustra::Migration::Manager.instance.refresh
```

Use `reinit!` when tests or external code may have changed the metadata table and you want the manager to recreate/check its internal state.

```crystal
Lustra::Migration::Manager.instance.reinit!
```
