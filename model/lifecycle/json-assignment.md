# JSON Assignment

Lustra models can be built and updated from JSON strings or IO objects.

## Build From JSON

Use `from_json` to build a new unsaved model:

```crystal
json = %({"first_name":"Ada","last_name":"Lovelace"})

user = User.from_json(json)
user.first_name # => "Ada"
user.persisted? # => false
```

`from_json` also accepts an `IO`:

```crystal
io = IO::Memory.new(%({"first_name":"Ada"}))
user = User.from_json(io)
```

## Create From JSON

Use `create_from_json` to build and save a model:

```crystal
user = User.create_from_json(%({"first_name":"Ada"}))
```

`create_from_json` returns the model. If validations fail, the model is returned
with errors and may not be persisted.

Use `create_from_json!` when validation failures should raise:

```crystal
user = User.create_from_json!(%({"first_name":"Ada"}))
```

## Update From JSON

Use `set_from_json` to assign JSON fields to an existing model without saving:

```crystal
user = User.find!(1)
user.set_from_json(%({"last_name":"Lovelace"}))
user.save!
```

Use `update_from_json` to assign and save:

```crystal
user.update_from_json(%({"last_name":"Lovelace"}))
```

`update_from_json` returns the model. If validations fail, the model is returned
with errors.

Use `update_from_json!` when validation failures should raise:

```crystal
user.update_from_json!(%({"last_name":"Lovelace"}))
```

## Nil Values

Nilable columns can be assigned `null` from JSON:

```crystal
user.set_from_json(%({"last_name":null}))
```

Non-nilable columns are not overwritten by `null` JSON values.

## Mass Assignment

Columns accept JSON assignment by default. Disable it per column with
`mass_assign: false`:

```crystal
class User
  include Lustra::Model

  column first_name : String
  column admin : Bool = false, mass_assign: false
end
```

With the default `trusted: false`, protected columns are ignored:

```crystal
user = User.from_json(%({"first_name":"Ada","admin":true}))
user.admin # => false
```

Pass `trusted: true` when the JSON source is trusted and protected columns
should be assigned:

```crystal
user = User.from_json(%({"first_name":"Ada","admin":true}), trusted: true)
user.admin # => true
```

Use `trusted: true` only for internal or already-authorized data. Do not use it
directly with untrusted request bodies.
