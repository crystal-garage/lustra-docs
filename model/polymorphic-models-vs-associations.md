# Polymorphic Models vs Associations

Lustra has two features with similar names:

| Feature | Database shape | Use when |
| :--- | :--- | :--- |
| Polymorphic associations | One child table stores `<name>_id` and `<name>_type` and can point to different parent tables. | A shared child record can belong to several parent model types. |
| Polymorphic models | Several Crystal model classes share one table and are distinguished by a type column. | One table stores different subclasses of the same logical record. |

This distinction is important for Rails users. Rails calls the first feature
"polymorphic associations". Rails usually calls the second pattern "single
table inheritance" or STI. Lustra uses the name "polymorphic models" for the
shared-table model feature.

## Polymorphic Associations

A polymorphic association keeps parent records in separate tables:

```text
employees
products
pictures
  imageable_id
  imageable_type
```

The child model belongs to one of several possible parent models:

```crystal
class Picture
  include Lustra::Model

  belongs_to imageable : Employee | Product, polymorphic: true
end
```

Use this for comments, pictures, attachments, reactions, audit entries, or any
other child record that can be attached to different parent tables.

See [Polymorphic Associations](associations/polymorphic-associations.md).

## Polymorphic Models

Polymorphic models keep different Crystal classes in one shared table:

```text
events
  type
  message
  user_id
  repository_id
```

The stored type decides which Crystal class should be instantiated:

```crystal
abstract class Event
  include Lustra::Model

  polymorphic through: "type"
end

class UserEvent < Event
end

class RepositoryEvent < Event
end
```

Use this when records are variants of one concept and mostly share the same
table structure.

See [Polymorphic Models](polymorphic-models.md).

## Quick Rule

If one row points to many possible parent tables, use a polymorphic association.

If many Crystal classes are stored in one table, use polymorphic models.
