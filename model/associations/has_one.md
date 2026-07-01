# `has_one`

`has_one` is used when another table stores a foreign key pointing to the current model, but only zero or one related record is expected.

Example:

```crystal
class User
  include Lustra::Model

  primary_key
  column email : String

  has_one profile : Profile
end

class Profile
  include Lustra::Model

  primary_key
  column display_name : String

  belongs_to user : User
end
```

`User#profile` returns `Profile?`:

```crystal
user = User.query.first!

if profile = user.profile
  puts profile.display_name
end
```

Use the bang variant when the related record must exist:

```crystal
profile = user.profile!
```

## Eager Loading

Collections get a generated `with_profile` helper:

```crystal
User.query.with_profile.each do |user|
  puts user.profile.try(&.display_name)
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
