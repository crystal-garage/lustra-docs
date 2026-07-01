# Converters

Converters translate values between PostgreSQL/Crystal DB values and model column types.

Lustra registers converters for common built-in types. Custom types need a converter registered with `Lustra::Model::Converter.add_converter`.

## Declare a Converter

A converter must provide:

```crystal
def self.to_column(value)
end

def self.to_db(value)
end
```

Example:

```crystal
struct MyApp::Color
  property r : UInt8 = 0
  property g : UInt8 = 0
  property b : UInt8 = 0
  property a : UInt8 = 255

  def self.from_string(value : String)
    # parse a database value
  end

  def to_s
    # serialize to a database value
  end
end

class MyApp::ColorConverter
  def self.to_column(value) : MyApp::Color?
    case value
    when Nil
      nil
    when MyApp::Color
      value
    else
      MyApp::Color.from_string(value.to_s)
    end
  end

  def self.to_db(value : MyApp::Color?)
    value.try(&.to_s)
  end
end

Lustra::Model::Converter.add_converter("MyApp::Color", MyApp::ColorConverter)
```

Then use the type in a model:

```crystal
class MyApp::Paint
  include Lustra::Model

  column id : Int64, primary: true, presence: false
  column color : MyApp::Color
end
```

When no `converter` option is given, Lustra looks up a converter from the Crystal type name.

## `converter` Option

Use the `converter` option when the converter name should not be inferred from the column type:

```crystal
Lustra::Model::Converter.add_converter("color_from_hex", MyApp::ColorConverter)

class MyApp::Paint
  include Lustra::Model

  column id : Int64, primary: true, presence: false
  column color : MyApp::Color, converter: "color_from_hex"
end
```

Converter lookup happens at compile time. If a converter is missing, Lustra raises a compile-time error explaining that no converter was found for the requested type/name.

## JSON Serializable Converter

For JSON serializable types, Lustra can generate a converter:

```crystal
class Actor
  include JSON::Serializable

  property name : String
end

Lustra.json_serializable_converter(Actor)

class Movie
  include Lustra::Model

  column id : Int64, primary: true, presence: false
  column actor : Actor
end
```

The generated converter stores the value as JSON and rebuilds the Crystal object when reading from the database.
