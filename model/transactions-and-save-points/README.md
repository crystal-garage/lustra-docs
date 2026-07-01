# Transactions And Savepoints

Lustra exposes transaction helpers through `Lustra::SQL`. Use them when multiple writes must succeed or fail as one unit.

```crystal
Lustra::SQL.transaction do
  user.update!(balance: user.balance - 100)
  recipient.update!(balance: recipient.balance + 100)
end
```

This section covers:

- `transaction` for atomic work
- `with_savepoint` for nested rollback points
- `rollback`, `rollback_transaction`, and `after_commit`
- connection pool behavior
