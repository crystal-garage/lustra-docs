# Select Clause

## The Select query

Lustra allows you to build Select query

## String substitution in SELECT

```ruby
Model.query.select(
  Lustra::SQL.raw("CASE WHEN x=:x THEN 1 ELSE 0 END as check", x: "blabla")
)
```

