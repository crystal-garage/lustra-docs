# Describing Your Columns

## `column`

Lustra models declare database-backed attributes with the `column` macro:

```crystal
column method_name : Type,
  primary: Bool = false,
  converter: String | Nil = nil,
  column_name: String | Nil = nil,
  presence: Bool = true,
  mass_assign: Bool = true
```

Common options:

| Option | Description |
| :--- | :--- |
| `primary` | Marks the column as the model primary key. Lustra expects one primary key for relations and default ordering. Compound primary keys are not supported. |
| `converter` | Selects a converter registered in `Lustra::Model::Converter`. When omitted, Lustra looks up a converter from the Crystal type. |
| `column_name` | Maps a Crystal method to a differently named PostgreSQL column. |
| `presence` | Controls validation/persistence presence checks. Use `presence: false` for non-null database-generated values such as `serial`/`bigserial` primary keys. |
| `mass_assign` | Controls whether the column can be assigned through mass-assignment paths such as JSON deserialization. |

## Different Column Names

Use `column_name` when the PostgreSQL column name does not match the Crystal method name:

```crystal
class User
  include Lustra::Model

  column id : Int64, primary: true, presence: false
  column login : String, column_name: "user_name"
end
```

`user.login` reads and writes the `user_name` database column.

## Uninitialized Columns

Lustra distinguishes `nil` from an unfetched value.

For example, this query only fetches two columns:

```crystal
User.query.select("first_name, last_name").each do |user|
  puts "#{user.first_name} #{user.last_name}"
end
```

Accessing a column that was not fetched raises an error:

```crystal
User.query.select("first_name, last_name").each do |user|
  user.id # raises because id was not initialized by this query
end
```

Use the generated column object when you need to test or handle this state:

```crystal
if user.id_column.defined?
  puts user.id
end
```

Column helpers:

| Helper | Description |
| :--- | :--- |
| `xxx_column.changed?` | Returns whether the column value changed since it was loaded or reset. |
| `xxx_column.has_db_default?` | Returns whether the column was declared with `presence: false`. |
| `xxx_column.name` | Returns the database column name. |
| `xxx_column.old_value` | Returns the previous value tracked by the column. |
| `xxx_column.revert` | Restores the previous value when one is available. |
| `xxx_column.clear` | Marks the column as undefined/unfetched. |
| `xxx_column.defined?` | Returns whether the column currently has a value. |
| `xxx_column.value` | Returns the value or raises if the column is undefined. |
| `xxx_column.value(default)` | Returns the current value or `default` when the column is undefined. |
| `xxx_column.change` | Returns `{old_value, new_value}` for changed columns when an old value is available. |

## Supported Types

Lustra registers converters for common PostgreSQL-backed Crystal types:

| Crystal | PostgreSQL |
| :--- | :--- |
| `String` | `text`, `varchar`, and other textual columns |
| `Bool` | `boolean` |
| `Int8`, `Int16`, `Int32`, `Int64` | integer columns |
| `UInt8`, `UInt16`, `UInt32`, `UInt64` | integer-compatible columns |
| `Float32`, `Float64` | floating-point columns |
| `BigInt`, `BigFloat`, `BigDecimal` | numeric-compatible columns |
| `UUID` | `uuid` |
| `JSON::Any` | `json`, `jsonb` |
| `Time` | timestamp/date-compatible values supported by the converter |
| `Array(Bool)`, `Array(String)`, `Array(Float32)`, `Array(Float64)`, `Array(Int32)`, `Array(Int64)` | PostgreSQL array columns |

Extensions register additional converters, such as enums, BCrypt passwords, time-in-day values, intervals, full-text search vectors, and PostgreSQL geometric types.

## BigDecimal and Numeric

`BigDecimal` maps naturally to PostgreSQL `numeric`. In migrations, use PostgreSQL numeric definitions directly when you need precision and scale:

```crystal
create_table(:payments) do |t|
  t.column "amount", "numeric(12, 2)"
end
```

PostgreSQL enforces numeric precision. If a value is too large for the declared precision, PostgreSQL raises an error during insert or update.
