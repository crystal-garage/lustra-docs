---
description: >-
  Associations describe links between Lustra models.
---

# Associations

Lustra supports the common Active Record style associations:

| Association | Meaning |
| :--- | :--- |
| `belongs_to` | The current model stores a foreign key pointing to another model. |
| `has_many` | The related model stores a foreign key pointing back to the current model. |
| `has_many through` | Models are connected through a join model/table. |
| `has_one` | The related model stores a foreign key, but only zero or one related record is expected. |
| Polymorphic associations | One child model can belong to one of several parent model types. |

Associations generate query helpers, assignment helpers, and eager-loading helpers such as `with_books`.

By default, Lustra follows naming conventions. You can override foreign keys and primary keys when your schema uses different names.

## Association Collections

Collection associations return query collections, not plain arrays. This means
you can refine an association query before executing it.

```crystal
author = Author.find!(1)

books = author
  .books
  .where { published_at != nil }
  .order_by(published_at: :desc)
  .limit(20)
```

The query is executed by terminal methods such as `each`, `to_a`, `first`,
`count`, or `empty?`.

```crystal
books.each do |book|
  puts book.title
end
```

Association collections can also be combined with scopes and eager-loading
helpers:

```crystal
books = author
  .books
  .with_reviews
  .with_count(:reviews, alias_name: "reviews_count")
  .order_by("books.id", :asc)
```

The same pattern works from `has_many through` associations:

```crystal
orders = book
  .orders
  .where(status: "paid")
  .order_by(date_submitted: :desc)
```

See [Polymorphic Associations](polymorphic-associations.md) when a child model can belong to more than one parent model type.
