# Connection Pool

Lustra opens database pools with `DB.open` and keeps a checked-out connection associated with the current fiber while a query is running.

```crystal
Lustra::SQL.init("postgres://localhost/my_app?initial_pool_size=1&max_pool_size=10")
```

You can register one default connection, one named connection, or several named connections.

```crystal
Lustra::SQL.init(ENV["DATABASE_URL"])

Lustra::SQL.init("analytics", ENV["ANALYTICS_DATABASE_URL"])

connections = {} of Lustra::SQL::Symbolic => String
connections["default"] = ENV["DATABASE_URL"]
connections["analytics"] = ENV["ANALYTICS_DATABASE_URL"]

Lustra::SQL.init(connections)
```

`add_connection` is an alias for registering another named pool.

```crystal
Lustra::SQL.add_connection("archive", ENV["ARCHIVE_DATABASE_URL"])
```

## Fiber Behavior

Outside a transaction, each SQL operation checks out a connection for the duration of that operation and returns it to the pool.

Inside a transaction, Lustra reuses the same connection for all SQL calls in the current fiber until the transaction commits or rolls back.

```crystal
Lustra::SQL.transaction do
  User.create!(email: "one@example.com")
  User.query.where(email: "one@example.com").first
end
```

Different fibers use separate checked-out connections. This matters for transaction visibility: a second fiber will not see uncommitted rows from the first fiber.

```crystal
spawn do
  Lustra::SQL.transaction do
    Lustra::SQL.insert(:events, {name: "queued"}).execute
    sleep 200.milliseconds
  end
end

spawn do
  sleep 100.milliseconds
  count = Lustra::SQL.select.from(:events).count
  # The insert above is still uncommitted in another fiber.
end
```

## Named Connections

Queries run on the `default` connection unless you select another one.

```crystal
Lustra::SQL.select.from(:events).use_connection("analytics").count
```

Transactions also accept a connection name.

```crystal
Lustra::SQL.transaction("analytics") do
  Lustra::SQL.select.from(:events).use_connection("analytics").count
end
```

Keep queries inside a named transaction on the same named connection. A query that uses another connection name will check out from another pool.

## Pool Size And Timeout

Pool options are configured in the PostgreSQL URL. Common options include:

```text
postgres://localhost/my_app?initial_pool_size=1&max_pool_size=10&checkout_timeout=5
```

Size the pool for the number of concurrent fibers that may hold a database connection at the same time. Long transactions and streaming result sets keep connections checked out longer, so they need more care than short single queries.
