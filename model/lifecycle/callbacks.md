# Callbacks

Callbacks let you run code before or after lifecycle events.

```crystal
class User
  include Lustra::Model

  column id : Int64, primary: true, presence: false
  column email : String
  column normalized_email : String?

  before(:validate) do |model|
    user = model.as(User)
    user.normalized_email = user.email.downcase
  end

  after(:create, :send_welcome_email)

  def send_welcome_email
    EmailManager.send_email(email)
  end
end
```

## Block Callbacks

Block callbacks receive `Lustra::Model`, so cast the argument before accessing model-specific methods:

```crystal
before(:validate) do |model|
  user = model.as(User)
  user.email = user.email.strip
end
```

## Method Callbacks

Use the method form when the callback should call a model method:

```crystal
after(:create, :send_welcome_email)
```

The method must be public.

## Events

Lustra supports callbacks around these events:

| Event | Triggered by |
| :--- | :--- |
| `:validate` | `valid?`, `valid!`, and save paths that validate. |
| `:save` | `save`, `save!`, and association save paths. |
| `:create` | Saving a new model with an `INSERT`. |
| `:update` | Saving a persisted model with changed columns. |
| `:destroy` | `destroy` and `destroy_all`. |

Direct lifecycle-bypassing methods such as `delete`, `delete_all`, `update_column`, `update_columns`, `update_all`, `increment!`, `decrement!`, and `touch` do not run model callbacks.
