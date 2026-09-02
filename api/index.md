# API reference

PrusaSlicer provides a global `api` table, a global `VolumeType` table, and several objects that PrusaSlicer creates and owns.
You do not construct those objects yourself.
API functions return them.

`api` is already available during command execution.
Do not call `require("api")`.

## Quick lookup

### Mesh creation and files

| Function                                                      | Returns | Summary                                                             |
| ------------------------------------------------------------- | ------- | ------------------------------------------------------------------- |
| [`api.make_cube`](geometry.md#apimake_cube)                   | `Mesh`  | Make an axis-aligned box.                                           |
| [`api.make_sphere`](geometry.md#apimake_sphere)               | `Mesh`  | Make a sphere.                                                      |
| [`api.make_cylinder`](geometry.md#apimake_cylinder)           | `Mesh`  | Make a cylinder.                                                    |
| [`api.make_cone`](geometry.md#apimake_cone)                   | `Mesh`  | Make a cone.                                                        |
| [`api.make_tetrahedron`](geometry.md#apimake_tetrahedron)     | `Mesh`  | Make a regular tetrahedron.                                         |
| [`api.make_prism`](geometry.md#apimake_prism)                 | `Mesh`  | Make a triangular prism.                                            |
| [`api.make_frustum`](geometry.md#apimake_frustum)             | `Mesh`  | Make a tapered cylinder whose top radius is half its bottom radius. |
| [`api.make_frustum_dowel`](geometry.md#apimake_frustum_dowel) | `Mesh`  | Make a faceted double-ended dowel.                                  |
| [`api.make_pyramid`](geometry.md#apimake_pyramid)             | `Mesh`  | Make a square pyramid.                                              |
| [`api.make_snap`](geometry.md#apimake_snap)                   | `Mesh`  | Make a two-sided snap connector.                                    |
| [`api.make_torus`](geometry.md#apimake_torus)                 | `Mesh`  | Make a torus.                                                       |
| [`api.load_stl`](geometry.md#apiload_stl)                     | `Mesh`  | Load an STL inside the plugin sandbox.                              |
| [`api.emboss_svg`](fonts-and-embossing.md#apiemboss_svg)      | `Mesh`  | Extrude SVG shapes into a mesh.                                     |
| [`api.emboss_text`](fonts-and-embossing.md#apiemboss_text)    | `Mesh`  | Turn text into a mesh.                                              |

### Fonts

| Function                                                             | Returns  | Summary                                                            |
| -------------------------------------------------------------------- | -------- | ------------------------------------------------------------------ |
| [`api.fonts`](fonts-and-embossing.md#apifonts)                       | `Font[]` | Get available fonts.                                               |
| [`api.get_default_font`](fonts-and-embossing.md#apiget_default_font) | `Font`   | Get the default font.                                              |
| [`api.get_font`](fonts-and-embossing.md#apiget_font)                 | `Font`   | Find the first case-sensitive substring match, or get the default. |

### Project

| Method                                                                                    | Returns        | Summary                                                        |
| ----------------------------------------------------------------------------------------- | -------------- | -------------------------------------------------------------- |
| [`api.project:add_object`](project.md#projectapiadd_object)                               | `ModelElement` | Add a mesh and optional extra volumes to the selected project. |
| [`api.project:current_bed`](project.md#projectapicurrent_bed)                             | `BedInstRef`   | Get the last selected bed.                                     |
| [`api.project:insert_layer_custom_gcode`](project.md#projectapiinsert_layer_custom_gcode) | nothing        | Append custom G-code at a Z height.                            |
| [`api.project:clear_layer_custom_steps`](project.md#projectapiclear_layer_custom_steps)   | nothing        | Remove all custom G-code entries from a bed.                   |

### Mesh and result types

| Type                                      | Members                                                                              |
| ----------------------------------------- | ------------------------------------------------------------------------------------ |
| [`ProjectApi`](project.md)                | `add_object`, `current_bed`, `insert_layer_custom_gcode`, `clear_layer_custom_steps` |
| [`Mesh`](geometry.md#mesh)                | `translate`, `bounds`                                                                |
| [`BoundingBox`](geometry.md#boundingbox)  | `min_x`, `min_y`, `min_z`, `max_x`, `max_y`, `max_z`                                 |
| [`ModelElement`](project.md#modelelement) | `type`                                                                               |
| [`Font`](fonts-and-embossing.md#font)     | `name`                                                                               |

### Beds, settings, and hardware

| Type                                                               | Members                                                                                        |
| ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| [`BedInstRef`](configuration-and-hardware.md#bedinstref)           | `printer_config`, `printer_presets`, `print_presets`, `tool_print_presets`, `material_presets` |
| [`ConfigBox`](configuration-and-hardware.md#configbox)             | `value`, `set`                                                                                 |
| [`HwPrinterConfig`](configuration-and-hardware.md#hwprinterconfig) | `name`, `tool_count`, `tools`                                                                  |
| [`HwToolConfig`](configuration-and-hardware.md#hwtoolconfig)       | `feature`, `nozzle_diameter`                                                                   |

## Global `VolumeType`

Use these named values for volumes inside `other_volumes`:

```lua
VolumeType.Solid
VolumeType.Negative
VolumeType.Modifier
VolumeType.SupportBlocker
VolumeType.SupportEnforcer
```

See [VolumeType](project.md#volumetype) for the meaning of each value.

## Types used in signatures

| Documentation type | Lua value                                                          |
| ------------------ | ------------------------------------------------------------------ |
| `boolean`          | `true` or `false`                                                  |
| `integer`          | A Lua integer.                                                     |
| `number`           | A Lua integer or float accepted wherever the API expects a number. |
| `string`           | A Lua string.                                                      |
| `table`            | A Lua table with the documented fields.                            |
| `T[]`              | A Lua-facing array or container of `T`, indexed from `1`.          |
| `T?`               | Optional. The value may be omitted or returned as `nil`.           |

Unless stated otherwise, distances use millimeters and angles use degrees.
