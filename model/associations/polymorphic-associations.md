# Polymorphic Associations

Polymorphic associations let one model belong to more than one parent model
through a pair of columns: `<name>_id` and `<name>_type`.

This is useful when several models can share the same kind of child record. For
example, both employees and products can have pictures.

## Schema

The child table stores the parent id and the parent type:

```sql
CREATE TABLE pictures (
  id bigserial PRIMARY KEY,
  name text NOT NULL,
  imageable_id bigint NOT NULL,
  imageable_type text NOT NULL
);
```

The value stored in `imageable_type` is the Crystal class name, such as
`Employee`, `Product`, or `Catalog::Product` for namespaced models.

## Models

Declare the polymorphic side with `belongs_to ..., polymorphic: true`. The
relation type must list the concrete parent model types.

```crystal
class Picture
  include Lustra::Model

  primary_key
  column name : String

  belongs_to imageable : Employee | Product, polymorphic: true
end
```

Declare each parent side with `has_many ..., as:`.

```crystal
class Employee
  include Lustra::Model

  primary_key
  column name : String

  has_many pictures : Picture, as: :imageable
end

class Product
  include Lustra::Model

  primary_key
  column name : String

  has_many pictures : Picture, as: :imageable
end
```

`belongs_to imageable` declares `imageable_id` and `imageable_type` columns on
`Picture`. You do not need to declare those columns manually in the model.

## Reading Child Records

Use the parent collection to read records for one concrete parent:

```crystal
employee = Employee.find!(1)

employee.pictures.each do |picture|
  puts picture.name
end
```

Lustra filters by both columns:

```sql
SELECT *
FROM pictures
WHERE imageable_id = 1
  AND imageable_type = 'Employee'
```

This means `Employee#1` and `Product#1` can both exist without their pictures
being mixed.

## Adding Records

Build or create records from the parent collection:

```crystal
picture = employee.pictures.build({name: "Profile photo"})
picture.save!

product.pictures.create!(name: "Product image")
```

Appending also sets both polymorphic columns and saves the record:

```crystal
picture = Picture.new({name: "Badge"})
employee.pictures << picture

picture.imageable_id   # => employee.id
picture.imageable_type # => "Employee"
```

Appending a persisted picture to a different parent moves it:

```crystal
product.pictures << picture

picture.imageable_id   # => product.id
picture.imageable_type # => "Product"
```

The parent must be persisted before accessing the polymorphic `has_many`
collection. Lustra raises a clear error if the parent primary key is not
defined.

## Reading the Parent

Use the polymorphic `belongs_to` reader to resolve the concrete parent:

```crystal
picture = Picture.find!(1)
parent = picture.imageable # Employee | Product
```

The reader uses `imageable_type` to choose the target model and
`imageable_id` to find the record.

Assigning a persisted parent updates both columns:

```crystal
picture.imageable = employee
picture.save!
```

## Eager Loading

Polymorphic `belongs_to` relations get a generated `with_*` helper:

```crystal
Picture.query.with_imageable.each do |picture|
  puts picture.imageable.name
end
```

Because the parent can live in different tables, eager loading runs one query
per declared target type. For `Employee | Product`, Lustra runs one query for
employees and one query for products, then stores the parents in the
association cache.

## Querying by Polymorphic Columns

Filter polymorphic records directly when you need a specific parent type:

```crystal
Picture.query.where(
  imageable_type: "Employee",
  imageable_id: employee.id
)
```

For namespaced models, use the full class name:

```crystal
Picture.query.where(imageable_type: "Catalog::Product")
```

## SQL Join Limitations

Polymorphic `belongs_to` associations cannot be auto-joined as a single SQL
table because the target can be one of several tables.

These helpers raise a clear error for polymorphic `belongs_to` associations:

```crystal
Picture.query.join(:imageable)
Picture.query.where.associated(:imageable)
Picture.query.where.missing(:imageable)
Picture.query.with_count(:imageable)
```

Use direct filters on `<name>_id` and `<name>_type`, or query each concrete
target type separately.

## Namespaced Models

Namespaced model classes are supported when the union uses the full type names:

```crystal
class Catalog::Picture
  include Lustra::Model

  belongs_to imageable : Catalog::Employee | Catalog::Product,
    polymorphic: true
end
```

Lustra stores the full class name, for example `Catalog::Employee`.
