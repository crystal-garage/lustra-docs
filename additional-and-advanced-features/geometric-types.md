# Geometric Types

Lustra supports PostgreSQL geometric types exposed by the `pg` shard.

Common model column types include:

- `PG::Geo::Point`
- `PG::Geo::Circle`
- `PG::Geo::Polygon`
- `PG::Geo::Box`
- `PG::Geo::Line`
- `PG::Geo::LineSegment`
- `PG::Geo::Path`

## Model Columns

Use the PostgreSQL type in the migration and the matching `PG::Geo` type in the model.

```crystal
create_table(:locations) do |t|
  t.column :name, :string, null: false
  t.column :coordinates, :point, null: false
  t.column :coverage_area, :circle
  t.column :service_boundary, :polygon
  t.column :bounding_box, :box
end
```

```crystal
class Location
  include Lustra::Model

  column name : String
  column coordinates : PG::Geo::Point
  column coverage_area : PG::Geo::Circle?
  column service_boundary : PG::Geo::Polygon?
  column bounding_box : PG::Geo::Box?
end
```

Assign geometric values with `PG::Geo` objects.

```crystal
Location.create!(
  name: "Downtown",
  coordinates: PG::Geo::Point.new(-74.0060, 40.7128),
  coverage_area: PG::Geo::Circle.new(-74.0060, 40.7128, 1000.0)
)
```

## Geometric Queries

Geometric helpers are available in the expression engine.

```crystal
target = PG::Geo::Point.new(-74.0060, 40.7128)

Location.query.where {
  coordinates.distance_from(target) <= 1000.0
}
```

Containment:

```crystal
point = PG::Geo::Point.new(-74.0, 40.7)

Location.query.where {
  coverage_area.contains?(point)
}
```

Other helpers map to PostgreSQL geometric operators:

| Helper | PostgreSQL operator |
| --- | --- |
| `distance_from`, `distance_to` | `<->` |
| `contains?` | `@>` |
| `contained_by?`, `within?` | `@>` with reversed operands |
| `overlaps?` | `&&` |
| `intersects?` | `?#` |
| `left_of?` | `<<` |
| `right_of?` | `>>` |
| `above?` | `|>>` |
| `below?` | `<<|` |
| `same_as?` | `~=` |

Convenience helpers combine distance with comparisons.

```crystal
Location.query.where { coordinates.within_distance?(target, 1000.0) }
Location.query.where { coordinates.closer_than?(target, 1000.0) }
Location.query.where { coordinates.farther_than?(target, 1000.0) }
```

## Indexes And Constraints

For spatial workloads, use PostgreSQL indexes and constraints that match your query patterns.

```crystal
create_table(:stores) do |t|
  t.column :delivery_area, :polygon
  t.index :delivery_area, using: :gist
end
```

Lustra also provides migration helpers for some geometric constraints.

```crystal
add_exclusion_constraint("stores", "delivery_area")
add_containment_check("stores", "delivery_area", "((0,0),(100,100))")
drop_constraint("stores", "stores_delivery_area_exclusion")
```
