# `has_one`

`has_one` is used when another table stores a foreign key pointing to the current model, but only zero or one related record is expected.

For example, each supplier can have one account:

```crystal
class Supplier
  include Lustra::Model

  primary_key
  column name : String

  has_one account : Account
end

class Account
  include Lustra::Model

  primary_key
  column account_number : String

  belongs_to supplier : Supplier
end
```

The foreign key lives on the `accounts` table. `Supplier#account` returns
`Account?`:

```crystal
supplier = Supplier.query.first!

if account = supplier.account
  puts account.account_number
end
```

Use the bang variant when the related record must exist:

```crystal
account = supplier.account!
```

## Eager Loading

Collections get a generated `with_account` helper:

```crystal
Supplier.query.with_account.each do |supplier|
  puts supplier.account.try(&.account_number)
end
```

## Options

```crystal
has_one relation_name : RelationType,
  foreign_key: "column_name",
  primary_key: "column_name"
```

| Option | Description | Default |
| :--- | :--- | :--- |
| `foreign_key` | Column stored on the related model. | current table singularized + `_id` |
| `primary_key` | Column on the current model matched against the foreign key. | model primary key |
