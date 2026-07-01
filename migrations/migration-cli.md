# Migration CLI

Lustra provides CLI commands for generating migration files and applying database migrations. Your application must load its database configuration and migration files in the CLI entrypoint you run.

## Generate A Migration

Use `lustra generate migration` to create a migration file under `src/db/migrations`.

```bash
lustra generate migration add_users
```

The generator creates a timestamped file like:

```text
src/db/migrations/202607010001_add_users.cr
```

Generated migration files include the migration module and an empty `change` method.

```crystal
class AddUsers
  include Lustra::Migration

  def change(dir)
    # TODO: Fill migration
  end
end
```

Use `--directory` or `-d` to generate files in another project directory.

```bash
lustra generate migration add_users --directory ./my_app
```

## Generate A Model And Migration

Use `lustra generate model` to create a model file and its first migration.

```bash
lustra generate model User email:string name:string
```

The generator writes files under:

```text
src/models/user.cr
src/db/migrations/<timestamp>_create_users.cr
```

Review generated migrations before running them, especially when you want custom indexes, constraints, or column options.

## Apply Migrations

Running `lustra migrate` with no subcommand applies all pending migrations.

```bash
lustra migrate
```

The explicit subcommand does the same thing.

```bash
lustra migrate migrate
```

## Migration Status

Use `status` to print all registered migrations and whether each one is applied.

```bash
lustra migrate status
```

## Move One Migration

Use `up` or `down` with a migration id to force one migration in that direction.

```bash
lustra migrate up 202607010001
lustra migrate down 202607010001
```

These commands fail when the migration is already in the requested state.

## Move To A Version

Use `set` to move the database to a target migration id.

```bash
lustra migrate set 202607010001
```

By default, `set` may apply and roll back migrations as needed. Use `--direction` to restrict the operation.

```bash
lustra migrate set 202607010001 --direction up
lustra migrate set 202607010001 --direction down
lustra migrate set 202607010001 --direction both
```

## Roll Back

Use `rollback` to roll back the last applied migration.

```bash
lustra migrate rollback
```

Pass a number to roll back more than one migration.

```bash
lustra migrate rollback 3
```

## Seeds

Use `seed` to run registered seed data.

```bash
lustra migrate seed
```
