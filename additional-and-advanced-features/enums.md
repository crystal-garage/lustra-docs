# Enums

Lustra can map PostgreSQL enum values to Crystal constants.

Define the enum in Crystal with `Lustra.enum`.

```crystal
Lustra.enum GenderType, "male", "female", "other" do
  def male?
    self == Male
  end
end
```

The macro creates constants from the string values.

```crystal
GenderType::Male.to_s
# => "male"

GenderType.all
# => [GenderType::Male, GenderType::Female, GenderType::Other]

GenderType.from_string("male")
# => GenderType::Male

GenderType.valid?("unknown")
# => nil
```

`from_string` raises `Lustra::IllegalEnumValueError` for unknown values.

## Migrations

Create the PostgreSQL enum type before creating columns that use it.

```crystal
class CreateUsers202607010001
  include Lustra::Migration

  def change(dir)
    create_enum(:gender_type, GenderType)

    create_table(:users) do |t|
      t.column :gender, :gender_type
    end
  end
end
```

You can also pass raw values.

```crystal
create_enum(:gender_type, ["male", "female", "other"])
```

Use `drop_enum` to remove a type. Pass the original values if the operation must be reversible.

```crystal
drop_enum(:gender_type, ["male", "female", "other"])
```

## Model Columns

Use the enum type in the model column declaration.

```crystal
class User
  include Lustra::Model

  column gender : GenderType
end
```

Assign enum constants in application code.

```crystal
user = User.new
user.gender = GenderType::Male
user.save!
```

Values loaded from PostgreSQL are converted back to the enum type.

```crystal
User.query.where { gender == GenderType::Female }.count
User.query.where { gender == "male" }.count
User.query.where { gender.in? GenderType.all }.count
```
