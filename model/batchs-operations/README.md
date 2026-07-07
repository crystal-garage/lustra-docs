# Batch Operations

Batch operations let you change many rows with fewer SQL queries.

Lustra provides two kinds of batch behavior:

- model-aware import with `Model.import`
- direct SQL inserts, upserts, updates, and deletes with `insert`, `insert_all`, `upsert`, `upsert_all`, `update_all`, `delete_all`, `to_update`, and `to_delete`

Use model-aware operations when validations and callbacks matter. Use direct SQL operations when you need speed and intentionally want to bypass model instances.
