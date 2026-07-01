# Batch Operations

Batch operations let you change many rows with fewer SQL queries.

Lustra provides two kinds of batch behavior:

- model-aware import with `Model.import`
- direct SQL updates/deletes with `update_all`, `delete_all`, `to_update`, and `to_delete`

Use model-aware operations when validations and callbacks matter. Use direct SQL operations when you need speed and intentionally want to bypass model instances.
