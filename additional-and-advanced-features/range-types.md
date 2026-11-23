# PostgreSQL Range Types

Lustra supports range columns using `PG::Range(T)`. They preserve empty ranges,
unbounded ends, and the inclusive/exclusive flags on both bounds.

Range support requires a crystal-pg version that provides `PG::Range`; Lustra's
`shard.yml` declares its PostgreSQL driver dependency.

## Supported Column Types

| Crystal model type | PostgreSQL column type |
| :--- | :--- |
| `PG::Range(Int32)` | `int4range` |
| `PG::Range(Int64)` | `int8range` |
| `PG::Range(BigDecimal)` | `numrange`, with exact decimal bounds |
| `PG::Range(PG::Numeric)` | `numrange`, with driver-native numeric bounds |
| `PG::Range(Time)` | `tsrange`, `tstzrange` |

Add `?` to the model type when the database column allows SQL `NULL`.

## Defining a Range Column

Use the PostgreSQL range type in the migration:

```crystal
create_table(:bookings) do |t|
  t.column "availability", "tsrange", null: true
end
```

Declare the matching model column:

```crystal
class Booking
  include Lustra::Model

  primary_key
  column availability : PG::Range(Time)?
end
```

Model construction accepts a native Crystal range:

```crystal
starts_at = Time.utc(2026, 12, 1, 9)
ends_at = Time.utc(2026, 12, 1, 17)
booking = Booking.create!(availability: starts_at...ends_at)

value = booking.availability.not_nil!
value.begin           # => starts_at
value.end             # => ends_at
value.lower_inclusive # => true
value.upper_inclusive # => false
value.empty?          # => false
```

## Inputs and Boundaries

Model construction, `set` with a hash, `update_column`, and `update_columns`
run the column converter. They accept native Crystal ranges, PostgreSQL range
strings, and `PG::Range` values.

Use `PG::Range` to express an exclusive lower bound:

```crystal
window = PG::Range(Time).new(
  starts_at,
  ends_at,
  lower_inclusive: false,
  upper_inclusive: true
)
booking.update!(availability: window)
```

PostgreSQL range text uses `[` or `]` for inclusive bounds and `(` or `)` for
exclusive bounds. Quoted ISO timestamps are accepted:

```crystal
booking.set({"availability" => "[\"2026-12-01 09:00:00\",\"2026-12-01 17:00:00\"]"})
booking.save!
```

Direct setters and keyword `set`, `update`, and `update!` calls require the
declared type, so pass a `PG::Range(Time)` or `nil`. For native ranges or strings,
use construction, hash-based `set` followed by `save!`, or a direct column update.
Direct column updates skip validations, callbacks, and automatic timestamp updates.

PostgreSQL canonicalizes integer ranges. For example, `[1,10]` is returned as
`[1,11)` after a database round trip; both contain the same integer values.
Continuous numeric and timestamp ranges retain their boundary flags.

## Querying Range Values

Use returned model values or native ranges in equality conditions:

```crystal
value = booking.availability.not_nil!
Booking.query.where { availability == value }
Booking.query.where { availability == (starts_at..ends_at) }
```

Lustra renders the values as PostgreSQL range literals. For low-level SQL
builders, pass `PG::Range` values directly:

```crystal
Lustra::SQL.insert(:bookings, {availability: window}).execute
```

## Empty, Unbounded, and Null Values

An empty range is a non-nil value that contains no elements:

```crystal
booking.update!(availability: PG::Range(Time).empty)
booking.availability.not_nil!.empty? # => true
```

An unbounded range is nonempty and has no finite endpoints:

```crystal
booking.set({"availability" => "(,)"})
booking.save!
value = booking.availability.not_nil!
value.empty? # => false
value.begin  # => nil
value.end    # => nil
```

Omit only one endpoint for a beginless or endless range, for example `"(,10]"`
or `"[1,)"` for an integer range column.

SQL `NULL` is represented by `nil`:

```crystal
booking.update!(availability: nil)
booking.availability # => nil
```

An empty string is invalid range input; use `"empty"`, `PG::Range(T).empty`, or
`nil` to express the intended state.

## Conversion and JSON

The model returns `PG::Range`, including when the input was a native Crystal
range. Keep this type for empty values or exclusive lower bounds. For a
nonempty value that Crystal can represent, call `to_crystal_range` explicitly:

```crystal
value = PG::Range(Int32).new(1, 10)
value.to_crystal_range # => 1...10
```

Range values serialize to JSON strings using PostgreSQL range syntax,
preserving empty values and boundary flags. Timestamp bounds are serialized
in UTC. Numeric bounds use exact decimal conversion without passing through
floating-point values.
