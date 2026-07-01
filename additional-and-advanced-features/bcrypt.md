# BCrypt

Lustra includes a converter for `Crypto::Bcrypt::Password`. Store the encrypted password in a text column and expose a writer that hashes plain text before assignment.

```crystal
class User
  include Lustra::Model

  primary_key "id", :uuid

  column encrypted_password : Crypto::Bcrypt::Password

  def password=(value : String)
    self.encrypted_password = Crypto::Bcrypt::Password.create(value)
  end
end
```

Migration:

```crystal
create_table(:users, id: :uuid) do |t|
  t.column :encrypted_password, :string, null: false
end
```

Create or update a password by assigning a freshly generated bcrypt password.

```crystal
user = User.create!({encrypted_password: Crypto::Bcrypt::Password.create("secret")})

user.encrypted_password.verify("secret")
# => true

user.encrypted_password.verify("wrong")
# => false
```

With the custom writer:

```crystal
user = User.new
user.password = "secret"
user.save!
```

Never store the plain text password in a model column.
