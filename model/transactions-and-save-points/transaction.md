# Transaction And Savepoints

Transactions make a group of database operations atomic. If the block raises, Lustra rolls back the transaction. If the block finishes normally, Lustra commits it.

```crystal
Lustra::SQL.transaction do
  sender.update!(balance: sender.balance - 100)
  recipient.update!(balance: recipient.balance + 100)
end
```

The block receives the checked-out database connection if you need it.

```crystal
Lustra::SQL.transaction do |connection|
  connection.exec("SELECT 1")
end
```

## Isolation Levels

`transaction` accepts an isolation level. The default is `Serializable`.

```crystal
Lustra::SQL.transaction(level: Lustra::SQL::Transaction::Level::ReadCommitted) do
  User.query.where(active: true).count
end

Lustra::SQL.transaction(level: Lustra::SQL::Transaction::Level::RepeatableRead) do
  User.query.where(active: true).count
end

Lustra::SQL.transaction(level: Lustra::SQL::Transaction::Level::Serializable) do
  User.query.where(active: true).count
end
```

PostgreSQL treats `READ UNCOMMITTED` as `READ COMMITTED`, so Lustra does not expose a separate `ReadUncommitted` value.

## Rollback

Call `Lustra::SQL.rollback` to roll back the current transaction without raising an application error outside the transaction block.

```crystal
Lustra::SQL.transaction do
  sender.update!(balance: sender.balance - 100)

  if recipient.locked?
    Lustra::SQL.rollback
  end

  recipient.update!(balance: recipient.balance + 100)
end
```

Use `rollback_transaction` when you are inside a savepoint but need to roll back the outer transaction.

```crystal
Lustra::SQL.transaction do
  Lustra::SQL.with_savepoint do
    Lustra::SQL.rollback_transaction
  end
end
```

## Nested Transactions

Calling `transaction` inside another `transaction` does not create a nested SQL transaction. Lustra reuses the current transaction connection and yields the inner block.

```crystal
Lustra::SQL.transaction do
  User.create!(email: "one@example.com")

  Lustra::SQL.transaction do
    User.create!(email: "two@example.com")
  end
end
```

If the inner block calls `Lustra::SQL.rollback`, the outer transaction is rolled back.

## Savepoints

Use `with_savepoint` when you need a nested rollback boundary.

```crystal
Lustra::SQL.transaction do
  User.create!(email: "one@example.com")

  Lustra::SQL.with_savepoint do
    User.create!(email: "two@example.com")
    Lustra::SQL.rollback
  end

  User.create!(email: "three@example.com")
end
```

In this example, the second user is rolled back to the savepoint, while the first and third users can still be committed.

You can name a savepoint when you need to target it explicitly.

```crystal
Lustra::SQL.with_savepoint(:before_import) do
  User.create!(email: "imported@example.com")
  Lustra::SQL.rollback(:before_import)
end
```

## After Commit

Use `after_commit` to register work that should run only after the surrounding transaction commits.

```crystal
Lustra::SQL.transaction do
  user.update!(confirmed: true)

  Lustra::SQL.after_commit do |_connection|
    ConfirmationEmailJob.enqueue(user.id)
  end
end
```

`after_commit` raises if it is called outside a transaction.
