# Fetching the Query

Collections are lazy until you fetch them.

Use model-fetching helpers when you want Lustra model instances:

```crystal
User.query.each do |user|
  puts user.email
end
```

Use SQL-fetching helpers when you want raw result hashes:

```crystal
User.query.select("id", "email").fetch do |row|
  puts "#{row["id"]}: #{row["email"]}"
end
```

For large datasets, use cursor-based fetching to avoid loading the full result set into memory at once.
