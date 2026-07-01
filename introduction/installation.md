# Setup

## Setup: In an existing project

```text
$ crystal init app <yourappname>
$ cd <yourappname>
```

### In `shards.yml`

Add Lustra to the dependencies list in `shards.yml`:

{% tabs %}
{% tab title="/shards.yml" %}
```yaml
dependencies:
  lustra:
    github: crystal-garage/lustra
    version: ">= 0.18.1"
```
{% endtab %}
{% endtabs %}

Then download the library:

{% tabs %}
{% tab title="terminal" %}
```text
$ shards install
```
{% endtab %}
{% endtabs %}

### In Your Source Code

Assuming your main entry point is `src/main.cr`, require Lustra and initialize the database connection:

{% tabs %}
{% tab title="src/main.cr" %}
```crystal
require "lustra"

Lustra::SQL.init("postgres://postgres@localhost/my_database")
```
{% endtab %}
{% endtabs %}

#### Connection URL

`require "lustra"` loads the library. `Lustra::SQL.init` registers a PostgreSQL connection pool.

The URL follows Crystal DB's PostgreSQL format:

```text
postgres://USER[:PASSWORD]@HOST/DATABASE[?*OPTIONS]
```

More information about the URL notation can be found [here](https://crystal-lang.org/docs/database/)

For production applications, configure the connection pool explicitly through URL parameters:

```crystal
Lustra::SQL.init("postgres://postgres@localhost/my_database?max_pool_size=10&initial_pool_size=1&max_idle_pool_size=2&checkout_timeout=5")
```

Tune `max_pool_size` per running process. For example, if you run two web processes with `max_pool_size=10` and two worker processes with `max_pool_size=5`, the application can open up to 30 PostgreSQL connections before counting migrations, monitoring, console sessions, or other services.

### Multiple Connections

You can register named connections:

```crystal
Lustra::SQL.init("readonly", "postgres://postgres@localhost/readonly_db?max_pool_size=5&initial_pool_size=1")
Lustra::SQL.init("primary", "postgres://postgres@localhost/primary_db?max_pool_size=10&initial_pool_size=1")
```

Models can use a named connection:

```crystal
class ReadOnlyModel
  include Lustra::Model

  self.connection = "readonly"

  column id : Int64, primary: true
  column data : String
end
```

### Installation Customization

You may want to install a smaller version of Lustra by requiring:

```crystal
require "lustra/core"
```

This loads Lustra without the built-in CLI and without some extensions \(JSONB, BCrypt, and others\).
