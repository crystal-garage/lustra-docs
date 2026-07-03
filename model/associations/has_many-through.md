# `has_many through`

`has_many through` connects two models through a join model/table.

Example schema:

```sql
CREATE TABLE books (
  id bigserial PRIMARY KEY,
  title text NOT NULL
);

CREATE TABLE orders (
  id bigserial PRIMARY KEY,
  date_submitted timestamp NOT NULL,
  status text NOT NULL
);

CREATE TABLE book_orders (
  id bigserial PRIMARY KEY,
  book_id bigint NOT NULL REFERENCES books(id) ON DELETE CASCADE,
  order_id bigint NOT NULL REFERENCES orders(id) ON DELETE CASCADE
);
```

Define a join model for the join table:

```crystal
class BookOrder
  include Lustra::Model

  primary_key

  belongs_to book : Book
  belongs_to order : Order
end
```

Then connect the main models through it:

```crystal
class Book
  include Lustra::Model

  primary_key
  column title : String

  has_many orders : Order, through: BookOrder
end

class Order
  include Lustra::Model

  primary_key
  column date_submitted : Time
  column status : String

  has_many books : Book, through: BookOrder
end
```

Now `book.orders` returns an `Order::Collection`:

```crystal
book = Book.query.first!

book.orders.each do |order|
  puts order.status
end
```

## Adding and Removing Links

Append creates the target record if needed and inserts the join row when it does not already exist:

```crystal
book = Book.query.first!
order = Order.create!(status: "paid", date_submitted: Time.local)

book.orders << order
```

Unlink deletes the join row:

```crystal
book = Book.query.first!
order = book.orders.where(status: "paid").first!

book.orders.unlink(order)
```

Call `unlink` on the relation collection (`book.orders`), not on a plain
`Order.query`, so Lustra knows which parent and through table to use.

## Autosave

By default, `build` creates an unsaved associated record and does not save it
when the parent is saved.

Use `autosave: true` when records built through the association should be saved
with the parent and linked through the join table.

```crystal
class Book
  include Lustra::Model

  primary_key
  column title : String

  has_many orders : Order, through: BookOrder, autosave: true
end

book = Book.new({title: "Lustra guide"})
book.orders.build({status: "draft", date_submitted: Time.local})
book.save!

book.orders.count # => 1
```

Autosave applies to records built through the association collection. Appending
with `<<` still saves or links the appended record immediately.

## `DISTINCT` and Advanced Queries

`has_many through` queries use `DISTINCT` to avoid returning the same target
record more than once when the join table can match multiple rows.

For normal association reads, keep the default:

```crystal
book.orders.each do |order|
  puts order.status
end
```

For advanced queries, you may need to remove that `DISTINCT` clause before
adding custom select fields, grouping, or ordering.

```crystal
orders = book
  .orders
  .clear_distinct
  .select(
    "orders.*",
    "COUNT(book_orders.id) AS books_count"
  )
  .group_by("orders.id")
  .order_by("books_count", :desc)
```

Use `clear_distinct` deliberately. If the join can produce duplicate target
rows, removing `DISTINCT` allows those duplicates to appear unless your query
handles them with grouping or another condition.

## Eager Loading

`has_many through` also generates a `with_*` helper:

```crystal
Book.query.with_orders.each do |book|
  book.orders.each do |order|
    puts order.status
  end
end
```

## Options

```crystal
has_many relation_name : RelationType,
  through: JoinModel,
  foreign_key: "target_id",
  own_key: "owner_id",
  autosave: true
```

| Option | Description | Default |
| :--- | :--- | :--- |
| `through` | Join model used to connect the two tables. | required |
| `foreign_key` | Join-table column pointing to the target model. | target table singularized + `_id` |
| `own_key` | Join-table column pointing to the current model. | current table singularized + `_id` |
| `autosave` | Saves built association records when the parent is saved. | `false` |

The join model does not need a primary key for simple join-table usage, but it should still map the join table accurately.
