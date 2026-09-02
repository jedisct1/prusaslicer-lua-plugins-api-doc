# Geometry and mesh API

[Back to the API index](index.md)

The geometry functions create or load a `Mesh` value.
You can transform the mesh data, inspect its bounding box, or pass it to `api.project:add_object`.

## Primitive overview

| Function             | Main dimensions               | Placement                                                           |
| -------------------- | ----------------------------- | ------------------------------------------------------------------- |
| `make_cube`          | X width, Y height, Z depth    | Starts at `(0, 0, 0)`.                                              |
| `make_sphere`        | Radius                        | Centered at the origin.                                             |
| `make_cylinder`      | Radius and Z height           | Centered on X/Y, base at Z 0.                                       |
| `make_cone`          | Base radius and Z height      | Centered on X/Y, base at Z 0.                                       |
| `make_tetrahedron`   | Edge length                   | Centered at the origin, with Y as its vertical axis.                |
| `make_prism`         | X width, Y length, Z height   | Centered on X/Y, base at Z 0.                                       |
| `make_frustum`       | Bottom radius and Z height    | Centered on X/Y, base at Z 0. Top radius is half the bottom radius. |
| `make_frustum_dowel` | Radius and half-height        | Centered at the origin, extending from `-height` to `+height` on Z. |
| `make_pyramid`       | Square base size and Z height | Centered on X/Y, base at Z 0.                                       |
| `make_snap`          | Radius and Z height           | Built around two separated snap halves.                             |
| `make_torus`         | Major and tube radii          | Centered at the origin in the X/Y plane.                            |

Primitive functions do not validate positive sizes or valid angle values.
Pass positive, finite dimensions.
Facet angles must also be positive.
Use at least `3` sectors for `make_frustum_dowel`.
Keep `make_snap`'s `space_proportion` between `0` and `0.5`, inclusive.

## `api.make_cube`

```lua
api.make_cube(width, height, depth) -> Mesh
```

Creates a box from `(0, 0, 0)` to `(width, height, depth)`.

```lua
local base = api.make_cube(40, 30, 3)
```

## `api.make_sphere`

```lua
api.make_sphere(radius, facet_angle?) -> Mesh
```

Creates a sphere centered at the origin.
`facet_angle` is an optional granularity angle in degrees and defaults to `1`.
Smaller angles generally produce more triangles.

```lua
local smooth = api.make_sphere(10)
local coarse = api.make_sphere(10, 12)
```

The sphere extends below Z 0.
Translate it upward before adding it if you want its bottom at Z 0:

```lua
local sphere = api.make_sphere(10)
sphere:translate(0, 0, 10)
```

## `api.make_cylinder`

```lua
api.make_cylinder(radius, height, facet_angle?) -> Mesh
```

Creates a cylinder centered on X/Y with its base at Z 0.
`facet_angle` defaults to `1` degree.

```lua
local peg = api.make_cylinder(3, 12, 10)
```

## `api.make_cone`

```lua
api.make_cone(radius, height, facet_angle?) -> Mesh
```

Creates a cone centered on X/Y with its circular base at Z 0 and tip at `height`.
`facet_angle` defaults to `1` degree.

```lua
local marker = api.make_cone(6, 15, 8)
```

## `api.make_tetrahedron`

```lua
api.make_tetrahedron(size) -> Mesh
```

Creates a regular tetrahedron with edge length `size`.
This primitive uses Y as its vertical direction and is centered around the origin.
Rotate it in the object definition if you need another orientation.

```lua
local tetrahedron = api.make_tetrahedron(20)

api.project:add_object {
    mesh = tetrahedron,
    rotate = {x = 90}
}
```

## `api.make_prism`

```lua
api.make_prism(width, length, height) -> Mesh
```

Creates a triangular prism.
The triangular cross-section spans `width` on X and reaches `height` on Z.
The prism spans `length` on Y.

```lua
local wedge = api.make_prism(20, 40, 10)
```

## `api.make_frustum`

```lua
api.make_frustum(radius, height, facet_angle?) -> Mesh
```

Creates a tapered cylinder centered on X/Y with its base at Z 0.
`radius` is the bottom radius.
The top radius is always half of it.
`facet_angle` defaults to `1` degree.

```lua
local tapered_peg = api.make_frustum(8, 20, 10)
```

## `api.make_frustum_dowel`

```lua
api.make_frustum_dowel(radius, height, sector_count) -> Mesh
```

Creates a faceted, double-ended dowel centered at the origin.
The mesh extends from `-height` to `+height` on Z, so its total Z size is twice the supplied height.
`sector_count` controls the number of sides around the middle.

```lua
local dowel = api.make_frustum_dowel(4, 6, 12)
dowel:translate(0, 0, 6)
```

## `api.make_pyramid`

```lua
api.make_pyramid(base, height) -> Mesh
```

Creates a square pyramid centered on X/Y.
The square base is at Z 0 and has side length `base`.

```lua
local pyramid = api.make_pyramid(20, 25)
```

## `api.make_snap`

```lua
api.make_snap(radius, height, space_proportion?, bulge_proportion?) -> Mesh
```

Creates the built-in two-sided snap connector shape.

- `space_proportion` controls the central spacing relative to `radius` and defaults to `0.25`.
- `bulge_proportion` controls the middle bulge relative to `radius` and defaults to `0.125`.

```lua
local snap = api.make_snap(4, 10)
local loose_snap = api.make_snap(4, 10, 0.3, 0.1)
```

## `api.make_torus`

```lua
api.make_torus(major_radius, tube_radius, major_facet_angle?, tube_facet_angle?) -> Mesh
```

Creates a torus centered at the origin in the X/Y plane.
Both optional facet angles use degrees and default to `1`.

```lua
local ring = api.make_torus(15, 2.5, 6, 12)
ring:translate(0, 0, 2.5)
```

## `api.load_stl`

```lua
api.load_stl(path) -> Mesh
```

Loads an STL file relative to the command entry script.
The resolved path must stay inside that entry script's directory.

```text
com.example.parts/
  manifest.json
  add_bracket.lua
  assets/
    bracket.stl
```

```lua
function execute(opts)
    local bracket = api.load_stl("assets/bracket.stl")

    api.project:add_object {
        mesh = bracket
    }
end
```

A missing, unreadable, empty, or unsafe file raises a Lua error.
PrusaSlicer runs its normal STL repair pass while loading, so some damaged meshes may still load successfully.

## `Mesh`

A `Mesh` holds triangle geometry outside the project.
Adding it to a project copies its data, so the original Lua value is not a live handle to the added object.

### `Mesh:translate`

```lua
mesh:translate(x, y, z)
```

Moves every vertex in the mesh and returns nothing.

```lua
local sphere = api.make_sphere(8)
sphere:translate(0, 0, 8)
```

This changes mesh geometry before insertion.
It is different from a `translate` field in an object or volume definition, which sets that volume's project transform.

### `Mesh:bounds`

```lua
mesh:bounds() -> BoundingBox
```

Returns the mesh's cached axis-aligned bounding box.

```lua
local model = api.load_stl("model.stl")
local bounds = model:bounds()
local width = bounds.max_x - bounds.min_x
```

In 3.0.0-alpha11, primitive constructors and `Mesh:translate` do not refresh the cached bounds.
Primitive bounds may be all zeroes, and bounds may stay stale after translation.
See [Primitive mesh bounds are stale](../known-limitations.md#primitive-mesh-bounds-are-stale).

## `BoundingBox`

`BoundingBox` has six read-only number fields:

```lua
bounds.min_x
bounds.min_y
bounds.min_z
bounds.max_x
bounds.max_y
bounds.max_z
```

Calculate sizes by subtraction:

```lua
local bounds = mesh:bounds()
local width = bounds.max_x - bounds.min_x
local depth = bounds.max_y - bounds.min_y
local height = bounds.max_z - bounds.min_z
```

[Back to the API index](index.md)
