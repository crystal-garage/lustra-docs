# Manage Migrations

Lustra migrations describe database schema changes as Crystal classes. A migration is registered when its class includes `Lustra::Migration`, so make sure migration files are required by your application or CLI entrypoint before running the migration manager.

```crystal
class CreateUsers202607010001
  include Lustra::Migration

  def change(dir)
    create_table(:users) do |t|
      t.column :email, :string, null: false, unique: true
      t.column :name, :string
      t.timestamps
    end
  end
end
```

Each migration runs inside a transaction. Lustra stores applied migration ids in the `__lustra_metadatas` table.

## Migration Ids

Lustra orders migrations by `uid`.

By default, the `uid` is discovered from the trailing number in the class name:

```crystal
class CreateUsers202607010001
  include Lustra::Migration

  def change(dir)
  end
end
```

If the class name has no trailing number, Lustra tries to read the leading number from the file name:

```text
202607010001_create_users.cr
```

You can also override `uid` manually.

```crystal
class CreateUsers
  include Lustra::Migration

  def uid : Int64
    202607010001_i64
  end

  def change(dir)
  end
end
```

Migration ids must be unique. Lustra raises an error when two registered migrations share the same id.

## Creating Tables

Use `create_table` to create a table. By default, Lustra adds an `id bigserial primary key` column.

```crystal
create_table(:users) do |t|
  t.column :email, :string, null: false, unique: true
  t.column :name, :string
  t.column :admin, :bool, default: false
  t.timestamps
end
```

Supported `id` options are `true`, `false`, `:bigserial`, `:serial`, and `:uuid`.

```crystal
create_table(:events, id: :uuid) do |t|
  t.column :name, :string, null: false
end

create_table(:memberships, id: false) do |t|
  t.column :user_id, :int64, null: false, primary: true
  t.column :team_id, :int64, null: false, primary: true
end
```

`t.column` maps common Crystal-style names to PostgreSQL types:

| Migration type | PostgreSQL type |
| --- | --- |
| `:string` | `text` |
| `:int32`, `:integer` | `integer` |
| `:int64`, `:long` | `bigint` |
| `:bigdecimal`, `:numeric` | `numeric` |
| `:datetime` | `timestamp without time zone` |

Other type names are passed through as SQL type names.

```crystal
t.column :metadata, :jsonb
t.column :tags, :string, array: true, index: "gin"
t.column :price, :numeric, null: false, default: "0"
t.column :name, :string, comment: "Public display name"
```

## References And Indexes

Use `references` for a foreign-key column and constraint.

```crystal
create_table(:posts) do |t|
  t.references to: "users", name: "user_id", on_delete: "cascade", null: false
  t.column :title, :string, null: false
end
```

Use `index` inside `create_table`.

```crystal
create_table(:users) do |t|
  t.column :email, :string, null: false
  t.index :email, unique: true
  t.index "lower(email)", using: :btree
end
```

Outside of `create_table`, use `create_index` or `add_index`.

```crystal
create_index "users", "email", unique: true
add_index "users", ["last_name", "first_name"]
```

## Changing Existing Tables

Use the column helpers for common `ALTER TABLE` operations.

```crystal
add_column "users", "nickname", "text", true
drop_column "users", "nickname", "text"
rename_column "users", "full_name", "name"
change_column_type "users", "age", "integer", "bigint"
change_column_null "users", "email", false, "''"
change_column_default "users", "role", {from: nil, to: "'member'"}
change_column_comment "users", "email", {from: nil, to: "Login email"}
```

Helpers are reversible when their operation exposes enough information for `down`. For one-way changes, use explicit direction blocks.

## Direction Blocks

The `dir` argument lets you run code only while migrating up or down.

```crystal
def change(dir)
  dir.up do
    execute "CREATE EXTENSION IF NOT EXISTS pgcrypto"
  end

  dir.down do
    execute "DROP EXTENSION IF EXISTS pgcrypto"
  end
end
```

If a migration cannot be rolled back, call `irreversible!` in the down direction.

```crystal
def change(dir)
  dir.up do
    execute "DELETE FROM audit_logs"
  end

  dir.down do
    irreversible!
  end
end
```

## Raw SQL

Use `execute` for SQL that is not covered by a helper.

```crystal
def change(dir)
  dir.up do
    execute "ALTER TABLE users ADD CONSTRAINT users_email_not_blank CHECK (email <> '')"
  end

  dir.down do
    execute "ALTER TABLE users DROP CONSTRAINT users_email_not_blank"
  end
end
```

Put raw SQL inside direction blocks unless the same statement is valid for both directions.
