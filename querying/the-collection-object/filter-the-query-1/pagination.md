# Pagination

Lustra includes `Lustra::SQL::Query::WithPagination`, which adds pagination
helpers to select builders and model collections.

Use `paginate` to fetch one page of records:

```crystal
per_page = 20
page = 3

posts = Post
  .query
  .where(published: true)
  .order_by(created_at: :desc)
  .order_by(id: :asc)
  .paginate(page: page, per_page: per_page)
```

This fetches page 3 with 20 records per page.

Always use a deterministic order for paginated queries. If the main ordering
column can have duplicate values, add a stable tie-breaker such as the primary
key.

```crystal
Post.query
  .order_by(created_at: :desc)
  .order_by(id: :asc)
```

## Total Count

`paginate` calculates the total count before applying `limit` and `offset`.
After calling it, the collection exposes pagination metadata:

```crystal
base = Post
  .query
  .where(published: true)

posts = base
  .order_by(created_at: :desc)
  .order_by(id: :asc)
  .paginate(page: page, per_page: per_page)

posts.total_entries # total matching records
posts.total_pages
posts.current_page
posts.per_page
posts.previous_page
posts.next_page
posts.first_page?
posts.last_page?
posts.out_of_bounds?
```

Because collections are mutable, `paginate` changes the collection by clearing
existing `limit` and `offset`, counting the records, and then applying the page
limit and offset. Use `dup` when you want to preserve a reusable base query:

```crystal
base = Post.query.where(published: true)

posts = base
  .dup
  .order_by(created_at: :desc)
  .order_by(id: :asc)
  .paginate(page: page, per_page: per_page)
```

## Page Bounds

Applications can use `out_of_bounds?` to detect a page past the end.

```crystal
raise "page out of range" if posts.out_of_bounds?
```

Use the same base query for pagination, so the total and the records are
calculated from the same filters.

## Association Pagination

Association collections can be paginated the same way:

```crystal
posts = user
  .posts
  .where(published: true)
  .order_by(created_at: :desc)
  .order_by(id: :asc)
  .paginate(page: page, per_page: per_page)
```

## Manual `limit` and `offset`

You can still use `limit` and `offset` directly when you do not need pagination
metadata:

```crystal
offset = (page - 1) * per_page

posts = Post
  .query
  .order_by(created_at: :desc)
  .order_by(id: :asc)
  .limit(per_page)
  .offset(offset)
```
