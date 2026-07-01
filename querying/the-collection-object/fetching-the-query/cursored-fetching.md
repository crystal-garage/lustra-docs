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
