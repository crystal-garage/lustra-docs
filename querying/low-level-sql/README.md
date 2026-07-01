# Writing low-level SQL

Under the hood, Lustra offers a performant SQL query builder for `SELECT`, `INSERT` and `DELETE` clauses:

 Under the hood, the

```ruby
Lustra::SQL.select.from("users").execute # SELECT * FROM users;
```



