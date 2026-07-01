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

Associations generate query helpers, assignment helpers, and eager-loading helpers such as `with_posts`.

By default, Lustra follows naming conventions. You can override foreign keys and primary keys when your schema uses different names.
