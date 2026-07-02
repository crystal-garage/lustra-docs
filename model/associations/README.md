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

Associations generate query helpers, assignment helpers, and eager-loading helpers such as `with_posts`.

By default, Lustra follows naming conventions. You can override foreign keys and primary keys when your schema uses different names.

## Association Collections

Collection associations return query collections, not plain arrays. This means
you can refine an association query before executing it.

```crystal
user = User.find!(1)

posts = user
  .posts
  .where(published: true)
  .order_by(created_at: :desc)
  .limit(20)
```

The query is executed by terminal methods such as `each`, `to_a`, `first`,
`count`, or `empty?`.

```crystal
posts.each do |post|
  puts post.title
end
```

Association collections can also be combined with scopes and eager-loading
helpers:

```crystal
posts = user
  .posts
  .published
  .with_tags
  .with_count(:tags, alias_name: "tags_count")
  .order_by("posts.id", :asc)
```

The same pattern works from `has_many through` associations:

```crystal
tags = post
  .tags
  .where { name =~ /orm/i }
  .order_by(name: :asc)
```

See [Polymorphic Associations](polymorphic-associations.md) when a child model can belong to more than one parent model type.
