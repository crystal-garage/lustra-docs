# Welcome to Lustra

Welcome to Lustra, an ORM built specifically for PostgreSQL and Crystal.

After reading this guide, you will know:

* How to install and configure Lustra for your project
* How to define models, columns, associations, validations, and lifecycle callbacks
* How to query PostgreSQL with Lustra collections and the lower-level SQL builder
* How to use PostgreSQL features such as JSONB, arrays, CTEs, cursors, enums, full-text search, and geometric types
* How to manage schema changes with migrations and keep database access predictable in production

## What is Lustra?

Lustra is an ORM (Object Relational Mapping) library for Crystal.

It provides an Active Record style model layer and a composable SQL builder for PostgreSQL applications.

Lustra is PostgreSQL-only. It is not designed for MariaDB, MySQL, or SQLite. This focus lets it expose PostgreSQL-specific features directly instead of hiding them behind a lowest-common-denominator API.

Lustra started as a fork of Clear at version 0.8. It is not compatible with later Clear releases. Since the fork, Lustra has evolved as an independent project with newer Crystal compatibility, expanded PostgreSQL support, additional tests, and behavior documented around production use.

Lustra is inspired by [Rails Active Record](https://github.com/rails/rails/tree/master/activerecord) and [Sequel](https://github.com/jeremyevans/sequel). Its main goals are:

* **Readable business code:** Model and query APIs should stay close to the application concepts they express.
* **PostgreSQL-first behavior:** JSONB, arrays, CTEs, cursors, full-text search, enums, UUIDs, and other PostgreSQL features are first-class tools.
* **Convention over configuration:** Common model, table, and foreign-key names work without extra setup, while explicit names are still supported when needed.

| Crystal | PostgreSQL |
| :--- | :--- |
| **class** ModelName | **TABLE** model\_names |
| **belongs\_to** foreign\_object | **COLUMN** foreign\_object\_id : bigint |

{% hint style="info" %}
If you already know Ruby on Rails, many naming conventions will feel familiar.
{% endhint %}
