# Array Querying

Lustra supports PostgreSQL array columns through model column converters and raw PostgreSQL array operators.

## Array Columns

Create array columns in migrations with `array: true`.

```crystal
create_table(:posts) do |t|
  t.column :tags_list, :string, array: true, index: "gin", default: "ARRAY['post']"
  t.column :flags, :bigint, array: true, index: "gin", default: "'{}'::bigint[]"
end
```

Declare the model column as an array.

```crystal
class Post
  include Lustra::Model

  column tags_list : Array(String)
  column flags : Array(Int64)
end
```

You can assign arrays normally.

```crystal
post.tags_list = ["crystal", "orm"]
post.flags = [1_i64, 2_i64]
post.save!
```

Empty arrays are supported when the Crystal type is explicit.

```crystal
Post.create!({title: "A post", tags_list: [] of String})
```

## IN Queries

Passing an array to `where` or `in?` creates an SQL `IN` condition for scalar columns.

```crystal
User.query.where(id: [1, 2, 3])

User.query.where { id.in?([1, 2, 3]) }
```

This is different from querying a PostgreSQL array column.

## PostgreSQL Array Operators

Use `raw` for PostgreSQL array-column operators such as `ANY`, `ALL`, `@>`, `<@`, and `&&`.

```crystal
Post.query.where { raw("? = ANY(tags_list)", "orm") }
```

`ANY` matches when one array element matches the value.

```crystal
Post.query
  .where { raw("? = ANY(tags_list)", "orm") }
  .pluck_col("title", String)
```

`ALL` matches when all array elements match the value.

```crystal
Post.query.where { raw("? = ALL(tags_list)", "orm") }
```

`@>` checks whether the column contains the given array.

```crystal
Post.query.where { raw("tags_list @> ARRAY[?]::text[]", "crystal") }
```

`<@` checks whether the column is contained by the given array.

```crystal
Post.query.where { raw("tags_list <@ ARRAY[?, ?]::text[]", "orm", "crystal") }
```

`&&` checks whether arrays overlap.

```crystal
Post.query.where { raw("tags_list && ARRAY[?, ?]::text[]", "sql", "crystal") }
```

Use explicit PostgreSQL casts such as `::text[]` or `::bigint[]` when PostgreSQL cannot infer the array type.
