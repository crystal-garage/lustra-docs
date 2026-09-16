# SQL Views and Read-Only Models

Lustra can work with PostgreSQL views in two related ways:

* map a view result to a read-only model
* register view definitions in code so migrations can recreate them in dependency order

## Read-Only Models

Use `self.read_only = true` when a model maps to a PostgreSQL view, a system catalog, or any relation that should not be saved through Lustra.

```crystal
class ActiveUserReport
  include Lustra::Model

  self.table = "active_user_reports"
  self.read_only = true

  column user_id : Int64, primary: true
  column email : String
  column posts_count : Int64
end
```

You can query a read-only model normally:

```crystal
ActiveUserReport.query
  .where { posts_count > 10 }
  .order_by(posts_count: "DESC")
  .each do |report|
    puts "#{report.email}: #{report.posts_count}"
  end
```

Saving is blocked:

```crystal
report = ActiveUserReport.query.first!
report.save  # => false
report.save! # raises Lustra::Model::ReadOnlyError
```

Lustra supports one primary key column per model. For a view-backed model, choose a result column that is unique for each row.

## Registering Views

`Lustra::View.register` lets you keep view definitions in Crystal code:

```crystal
Lustra::View.register :active_user_reports do |view|
  view.query <<-SQL
    SELECT
      users.id AS user_id,
      users.email,
      COUNT(posts.id) AS posts_count
    FROM users
    LEFT JOIN posts ON posts.user_id = users.id
    WHERE users.active = TRUE
    GROUP BY users.id, users.email
  SQL
end
```

Registration stores the definition in the running program; it does not execute SQL. After initializing the connection, creating the source tables, and loading the definitions, create the views:

```crystal
Lustra::View.apply(:create)

rows = Lustra::SQL.select.from("public.active_user_reports").to_a
reports = ActiveUserReport.query.to_a
```

`Lustra::Migration::Manager.instance.apply_all` drops registered views, applies pending migrations, and recreates the views from their loaded definitions. This also happens when no migrations are pending. Individual `up`, `down`, and `apply_to` calls do not perform this view lifecycle. Load view definitions in the migration entry point before calling `apply_all`.

## View Dependencies

If a view depends on another registered view, declare the dependency with `require`:

```crystal
Lustra::View.register :daily_post_counts do |view|
  view.query <<-SQL
    SELECT user_id, DATE(created_at) AS day, COUNT(*) AS posts_count
    FROM posts
    GROUP BY user_id, DATE(created_at)
  SQL
end

Lustra::View.register :active_user_daily_post_counts do |view|
  view.require(:daily_post_counts)

  view.query <<-SQL
    SELECT users.id AS user_id, daily_post_counts.day, daily_post_counts.posts_count
    FROM users
    INNER JOIN daily_post_counts ON daily_post_counts.user_id = users.id
    WHERE users.active = TRUE
  SQL
end
```

Dependencies are created first and dropped last, regardless of registration order. Cyclic dependencies raise `ArgumentError`.

## Schema and Connection

By default, views are created in the `public` schema on the default connection.

```crystal
Lustra::View.register :admin_reports do |view|
  view.schema :reporting
  view.connection "primary"
  view.query "SELECT * FROM reports"
end
```

## Materialized Views

`materialized(true)` is available:

```crystal
Lustra::View.register :expensive_report do |view|
  view.materialized true
  view.query "SELECT * FROM reports"
end
```

Create the materialized view once after registering it, then query its stored results:

```crystal
Lustra::View.apply(:create)
rows = Lustra::SQL.select.from("public.expensive_report").to_a
```

Lustra does not refresh materialized views automatically. Refresh them with SQL when their source data changes:

```crystal
Lustra::SQL.execute("REFRESH MATERIALIZED VIEW public.expensive_report")
```

A refresh updates stored data; it does not change the view definition. After changing a definition, use the migration manager's drop-and-create lifecycle. Calling `apply(:create)` again does not replace an existing materialized view.

For a view registered on a named connection, use that connection for both reads and refreshes:

```crystal
Lustra::SQL.select.from("reporting.expensive_report").use_connection("primary").to_a
Lustra::SQL.execute("primary", "REFRESH MATERIALIZED VIEW reporting.expensive_report")
```

This last example assumes the view was registered with `view.schema :reporting` and `view.connection "primary"`, and that the schema already exists.

