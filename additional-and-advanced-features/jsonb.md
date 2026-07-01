# JSONB

Lustra supports PostgreSQL `jsonb` columns through normal model columns and expression helpers.

```crystal
class User
  include Lustra::Model

  column notification_preferences : JSON::Any
end
```

In migrations, create a `jsonb` column and usually add a GIN index when you query it often.

```crystal
create_table(:users) do |t|
  t.column :notification_preferences, :jsonb, index: "gin", default: "'{}'"
end
```

## JSONB Paths

Use `jsonb(path)` in query expressions. Dot-separated paths are converted to PostgreSQL JSONB operators.

```crystal
User.query.where { notification_preferences.jsonb("email.enabled") == true }
```

For literal equality, Lustra uses the index-friendly `@>` containment form.

```sql
WHERE "notification_preferences" @> '{"email":{"enabled":true}}'
```

Escape a literal dot in a key with a backslash.

```crystal
User.query.where { notification_preferences.jsonb("external\\.id") == "abc" }
```

## Casting

Use `cast` when you need to compare a JSONB path as another SQL type.

```crystal
User.query.where {
  notification_preferences.jsonb("email.frequency").cast("text") == "daily"
}
```

Casting uses arrow notation instead of the containment form.

## Key Existence

Use the JSONB key helpers for PostgreSQL `?`, `?|`, and `?&`.

```crystal
User.query.where { notification_preferences.jsonb_key_exists?("email") }

User.query.where {
  notification_preferences.jsonb_any_key_exists?(["email", "sms"])
}

User.query.where {
  notification_preferences.jsonb_all_keys_exists?(["email", "sms"])
}
```

You can call key helpers on nested JSONB paths too.

```crystal
User.query.where {
  notification_preferences.jsonb("channels").jsonb_key_exists?("email")
}
```

## JSONB Arrays

Use `contains?` on a JSONB path to test whether an array contains a value.

```crystal
User.query.where {
  notification_preferences.jsonb("enabled_channels").contains?("email")
}
```

## Low-Level Helpers

`Lustra::SQL::JSONB` exposes helper methods when you need SQL fragments outside the expression engine.

```crystal
include Lustra::SQL::JSONB

jsonb_exists?("notification_preferences", "email")
jsonb_any_exists?("notification_preferences", ["email", "sms"])
jsonb_all_exists?("notification_preferences", ["email", "sms"])
jsonb_eq("notification_preferences", "email.enabled", true)
```
