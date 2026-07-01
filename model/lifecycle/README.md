# Lifecycle

The model lifecycle covers the work Lustra performs around persistence:

| Area | Purpose |
| :--- | :--- |
| Persistence | Insert, update, reload, delete, and destroy records. |
| Validations | Decide whether a model can be saved. |
| Callbacks | Run application code before or after lifecycle events. |

Most model-level write methods go through validations and callbacks. Some direct SQL helpers intentionally bypass the lifecycle for performance or atomic updates; those are documented in [Persistence](persistence.md).
