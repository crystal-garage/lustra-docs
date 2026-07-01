# Handling Multiple Connections

Lustra uses the `default` connection unless a model or query selects another connection.

## Register Connections

Register the default connection with one URL.

```crystal
Lustra::SQL.init(ENV["DATABASE_URL"])
```

Register a named connection with `init(name, url)` or `add_connection`.

```crystal
Lustra::SQL.init("secondary", ENV["SECONDARY_DATABASE_URL"])

Lustra::SQL.add_connection("archive", ENV["ARCHIVE_DATABASE_URL"])
```

You can also initialize several connections from a hash.

```crystal
Lustra::SQL.init({
  "default" => ENV["DATABASE_URL"],
  "legacy" => ENV["LEGACY_DATABASE_URL"],
})
```

## Model Connection

Set `self.connection` on a model to make all model queries use a named connection.

```crystal
class PostStat
  include Lustra::Model

  self.connection = "secondary"
  self.table = "post_stats"

  column id : Int32, primary: true, presence: false
  column post_id : Int32
end
```

Models without an explicit connection use `default`.

```crystal
Post.connection
# => "default"

PostStat.connection
# => "secondary"
```

## Low-Level Queries

Use `use_connection` on low-level SQL builders.

```crystal
Lustra::SQL.select
  .from(:post_stats)
  .use_connection("secondary")
  .fetch do |row|
    puts row["post_id"]
  end
```

Transactions accept a connection name.

```crystal
Lustra::SQL.transaction("secondary") do
  PostStat.query.create!(post_id: 1)
end
```

Keep all queries inside a named transaction on the same connection. A query that selects a different connection name uses another pool and is not part of that transaction.

## Migrations

The migration manager runs on the default connection. If you need schema changes on another database, run a separate migration setup against that database.

Avoid associations between models stored in different databases. Lustra cannot make cross-database joins work unless PostgreSQL itself can see both relations through the same connection.
