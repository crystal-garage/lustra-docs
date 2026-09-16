# Cursor Fetching

Normal fetching can load the full result set into memory.

For large result sets, use PostgreSQL cursor-based fetching.

## Model Cursor Fetching

`each_with_cursor` yields model instances in batches:

```crystal
User.query.each_with_cursor do |user|
  puts user.email
end
```

Set the batch size with the first argument:

```crystal
User.query.each_with_cursor(100) do |user|
  puts user.email
end
```

You can also pass `fetch_columns`:

```crystal
User.query
  .select("users.*", "LOWER(email) AS normalized_email")
  .each_with_cursor(500, fetch_columns: true) do |user|
    puts user["normalized_email"]
  end
```

## Raw Cursor Fetching

`fetch_with_cursor` yields raw result hashes:

```crystal
User.query.select("id", "email").fetch_with_cursor(count: 500) do |row|
  puts "#{row["id"]}: #{row["email"]}"
end
```

Cursor fetching runs inside a transaction because PostgreSQL cursors are transaction-scoped.

## Batch Sizes and Cleanup

If the collection is already cached, `each_with_cursor` yields the cached models without opening a cursor.

When opening a cursor, batch sizes must be positive. Zero or negative values raise `ArgumentError`; low-level `fetch_with_cursor` validates the size before query hooks or SQL run.

Lustra explicitly closes the cursor after normal completion, an early exit, or a callback exception. This also applies when iteration reuses an outer transaction, so a finished cursor does not stay open until that transaction ends.

```crystal
Lustra::SQL.transaction do
  User.query.order_by(:id).each_with_cursor(batch: 100) do |user|
    puts user.email
  end

  # The cursor is closed, while the outer transaction remains open.
end
```

Cleanup preserves the original iteration exception if the transaction is aborted or the connection is lost. Cursor iteration uses the model or query's selected connection.
