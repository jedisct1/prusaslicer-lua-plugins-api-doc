# Project API

[Back to the API index](index.md)

`api.project` is bound to the project selected when the command starts.
Its methods add objects and work with the selected bed's custom G-code.
The selected project is the project currently active in the PrusaSlicer interface.
Its selected bed is the build plate currently selected there.

## `ProjectApi:add_object`

```lua
api.project:add_object(definition) -> ModelElement
```

Adds one object to the project.
The definition must contain a main `mesh` and may contain transforms, settings, and extra volumes.

### Object definition

```lua
{
    mesh = Mesh,
    translate = {x = number?, y = number?, z = number?}?,
    rotate = {x = number?, y = number?, z = number?}?,
    params = table<string, value>?,
    object_params = table<string, value>?,
    other_volumes = VolumeDefinition[]?
}
```

| Field | Required | Meaning |
|---|---:|---|
| `mesh` | Yes | Main solid mesh. |
| `translate` | No | Main volume offset in millimeters. Missing axes default to 0. |
| `rotate` | No | Main volume rotation in degrees. Missing axes default to 0. |
| `params` | No | Settings applied to the main volume. |
| `object_params` | No | Settings applied to the whole object. |
| `other_volumes` | No | Extra solid, negative, modifier, or support volumes. |

The main mesh is always created as a solid model part.
A top-level `type` field is ignored in alpha11.

If a bed is selected, PrusaSlicer centers the new object's exact X/Y bounding box on that bed.
Only printable solid model-part volumes contribute to this box.
Negative, modifier, support blocker, and support enforcer volumes do not.

For a single-volume object, this automatic centering cancels the main volume's top-level X/Y translation.
With several solid volumes, their relative X/Y layout is kept while the whole printable group is centered.
The mesh geometry and volume transforms still control its Z position.

```lua
local model = api.make_cube(30, 20, 8)

local element = api.project:add_object {
    mesh = model,
    translate = {z = 0},
    object_params = {
        fill_density = "15%"
    }
}
```

Settings are applied through the limited configuration setter.
See [ConfigBox and configuration values](configuration-and-hardware.md#configbox) before relying on a key.

### Volume definition

Each item in `other_volumes` has this shape:

```lua
{
    mesh = Mesh,
    translate = {x = number?, y = number?, z = number?}?,
    rotate = {x = number?, y = number?, z = number?}?,
    type = VolumeType?,
    params = table<string, value>?
}
```

If `type` is missing:

- A volume with a `params` field becomes `VolumeType.Modifier`, even when `params` is empty.
- A volume without a `params` field becomes `VolumeType.Solid`.

Set the type explicitly when the role matters.

### Multi-volume example

This example adds a solid base, a negative cylinder, and a modifier region:

```lua
function execute(opts)
    local base = api.make_cube(40, 30, 8)
    local hole = api.make_cylinder(4, 8, 8)
    local modifier = api.make_cube(20, 30, 4)

    api.project:add_object {
        mesh = base,
        other_volumes = {
            {
                mesh = hole,
                type = VolumeType.Negative,
                translate = {x = 20, y = 15}
            },
            {
                mesh = modifier,
                type = VolumeType.Modifier,
                translate = {x = 10, z = 4},
                params = {
                    fill_density = "60%"
                }
            }
        }
    }
end
```

Transforms belong to each volume.
They do not move every volume in the object as a group.

## `VolumeType`

`VolumeType` is a global read-only enum table.
Use its names instead of numeric values.

| Value | Meaning |
|---|---|
| `VolumeType.Solid` | Printable solid model part. |
| `VolumeType.Negative` | Subtracts intersecting geometry. |
| `VolumeType.Modifier` | Changes settings in the intersecting region. |
| `VolumeType.SupportBlocker` | Prevents support generation in the region. |
| `VolumeType.SupportEnforcer` | Requests support generation in the region. |
| `VolumeType.Invalid` | Invalid or uninitialized role. Do not use it in a definition. |

Numeric enum values are not part of the documented contract.

### Support volume example

```lua
local blocker = api.make_cube(10, 10, 15)

api.project:add_object {
    mesh = api.load_stl("part.stl"),
    other_volumes = {
        {
            mesh = blocker,
            type = VolumeType.SupportBlocker,
            translate = {x = 5, y = 5}
        }
    }
}
```

## `ModelElement`

`add_object` currently returns a handle with one read-only property:

```lua
element.type -> integer
```

`type` is a numeric category code.
There is no public table of those codes, no other `ModelElement` method, and no current API that consumes the handle.

## `ProjectApi:current_bed`

```lua
api.project:current_bed() -> BedInstRef
```

Returns the last selected bed in the active project.

```lua
local bed = api.project:current_bed()
local printer = bed:printer_config()
print(printer.name)
```

The implementation assumes that a valid bed is selected.
Use this method from the normal Plater command flow.

See [Configuration and hardware](configuration-and-hardware.md) for `BedInstRef` methods.

## `ProjectApi:clear_layer_custom_steps`

```lua
api.project:clear_layer_custom_steps(bed)
```

Clears the bed's entire custom G-code list.
It returns nothing.
If the bed has no list yet, the call does nothing.
An invalid bed reference raises an error.

```lua
local bed = api.project:current_bed()
api.project:clear_layer_custom_steps(bed)
```

This removes every custom entry for that bed, including entries the user created before running the plugin.
Ask for confirmation through your own parameter design if that data matters.

## `ProjectApi:insert_layer_custom_gcode`

```lua
api.project:insert_layer_custom_gcode(bed, z_depth, gcode)
```

Appends one custom G-code entry to `bed`.

| Parameter | Type | Meaning |
|---|---|---|
| `bed` | `BedInstRef` | Target bed. |
| `z_depth` | number | Print Z height in millimeters. |
| `gcode` | string | Raw G-code text. |

The entry uses extruder number 1.
The method appends without sorting the existing list, so add entries in ascending Z order.

```lua
local bed = api.project:current_bed()

api.project:clear_layer_custom_steps(bed)
api.project:insert_layer_custom_gcode(bed, 5.2, "M104 S220")
api.project:insert_layer_custom_gcode(bed, 15.2, "M104 S210")
```

PrusaSlicer does not validate the G-code string for the plugin.
Generate commands that are safe for the selected printer and firmware.

[Back to the API index](index.md)
