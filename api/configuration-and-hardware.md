# Configuration and hardware API

[Back to the API index](index.md)

Start with the selected bed:

```lua
local bed = api.project:current_bed()
```

The bed gives you access to selected presets and the evaluated printer hardware description.

## Indexing rules

This API currently uses two indexing conventions:

- Lua-facing arrays such as `printer.tools` start at `1`.
- Indices passed to `tool_print_presets` and `material_presets` start at `0`.

For the first tool or material slot:

```lua
local printer = bed:printer_config()
local first_tool = printer.tools[1]

local first_tool_print = bed:tool_print_presets(0)
local first_material = bed:material_presets(0)
```

Passing an out-of-range preset index raises an error.

## `BedInstRef`

`BedInstRef` is a reference to one bed instance in the active project.
It has no public constructor.

### `BedInstRef:printer_config`

```lua
bed:printer_config() -> HwPrinterConfig
```

Returns the evaluated hardware configuration for the bed's selected printer.

```lua
local printer = bed:printer_config()
print(printer.name, printer.tool_count)
```

### `BedInstRef:printer_presets`

```lua
bed:printer_presets() -> ConfigBox
```

Returns the selected printer preset values.

### `BedInstRef:print_presets`

```lua
bed:print_presets() -> ConfigBox
```

Returns the selected print preset values.

```lua
local print_settings = bed:print_presets()
local layer_height = print_settings:value("layer_height")
```

### `BedInstRef:tool_print_presets`

```lua
bed:tool_print_presets(tool_index) -> ConfigBox
```

Returns tool-specific print settings.
`tool_index` is zero-based.

```lua
local first_tool_settings = bed:tool_print_presets(0)
```

### `BedInstRef:material_presets`

```lua
bed:material_presets(slot_index) -> ConfigBox
```

Returns the selected material preset for a slot.
`slot_index` is zero-based.

```lua
local first_material = bed:material_presets(0)
first_material:set("filament_max_volumetric_speed", 12)
```

## `ConfigBox`

A `ConfigBox` is a live view of one selected preset's configuration values.

It has two methods:

```lua
config:value(name) -> value
config:set(name, value)
```

There is no method to list keys, inspect a key's type, reset a value, or ask whether `set` succeeded.

### `ConfigBox:value`

```lua
config:value(name) -> boolean|integer|number|string|nil|opaque value
```

For supported settings, this returns the current value.

```lua
local print_settings = api.project:current_bed():print_presets()
local height = print_settings:value("layer_height")
print("Layer height: " .. height)
```

Directly usable return types are:

- Boolean
- Integer
- Optional integer, returned as an integer or `nil`
- Number
- String

Some complex setting values are returned as opaque values that Lua code cannot inspect.
These include 2D vectors, percentages, and values that may be either a number or a percentage.

An unknown name raises this kind of error:

```text
Invalid preset item name 'unknown_name': not found
```

A known setting with an unsupported getter type raises `Unsupported config type`.

### `ConfigBox:set`

```lua
config:set(name, value)
```

Changes a supported setting and returns nothing.

The current setter supports these destination types:

| Destination kind     | Lua input                                    | Example                  |
| -------------------- | -------------------------------------------- | ------------------------ |
| Integer              | number                                       | `3`                      |
| Number               | number                                       | `0.2`                    |
| Percentage           | number or numeric string, with optional `%`  | `15`, `"15"`, or `"15%"` |
| Number or percentage | number, numeric string, or percentage string | `0.4` or `"120%"`        |
| Enum                 | serialized string                            | `"outer_only"`           |

An enum's serialized string is the text stored in configuration files, such as `outer_only`.
It may differ from the translated label shown in the interface.

Pass whole numbers to integer settings.
A fractional input is rounded to the nearest integer, with halfway values rounded away from zero.

Examples:

```lua
local bed = api.project:current_bed()
local print_settings = bed:print_presets()

print_settings:set("layer_height", 0.2)
print_settings:set("fill_density", "20%")
print_settings:set("brim_type", "outer_only")
print_settings:set("brim_width", 5)
```

The setter does not currently support boolean, ordinary string, vector, or many list destination types.
An unknown key does nothing.

A known setting with an unsupported type keeps its stored value.
If that setting is an override, alpha11 can still enable the override.
An invalid enum string is written to the log and also leaves the stored value unchanged, but it has the same override-enabling side effect.
`ConfigBox:set` returns no status for any of these cases.

The setter is also used for `object_params` and volume `params` passed to `api.project:add_object`.
Those two paths log a misleading `not found` error for both an unknown key and a known unsupported type.

When a readable setting matters, verify it after writing:

```lua
print_settings:set("layer_height", 0.24)
local actual = print_settings:value("layer_height")
assert(math.abs(actual - 0.24) < 1e-9, "Could not set layer_height")
```

Compare floating-point values with a tolerance rather than exact equality.


## Finding configuration keys

Configuration names are PrusaSlicer setting keys, not translated UI labels.
For example, the UI's layer height field uses `layer_height`.

PrusaSlicer can export its current configuration schema:

```sh
PrusaSlicer --export-config-schema config-schema.json
```

The JSON describes printer, print, tool-print, filament, and SLA configuration models.
Use it to find serialized names, types, and enum values for the exact build you support.

The schema describes PrusaSlicer settings in general.
It does not mean every type can be read or written through the current Lua binding.
Cross-check the supported types above and test the specific key.

Useful keys demonstrated by bundled plugins include:

| Preset   | Key                                  | Kind                 | Example value  |
| -------- | ------------------------------------ | -------------------- | -------------- |
| Print    | `layer_height`                       | number               | `0.2`          |
| Print    | `first_layer_height`                 | number or percentage | `0.2`          |
| Print    | `fill_density`                       | percentage           | `"15%"`        |
| Print    | `brim_type`                          | enum                 | `"outer_only"` |
| Print    | `brim_width`                         | number               | `5`            |
| Material | `filament_max_volumetric_speed`      | number               | `12`           |
| Volume   | `perimeters`                         | integer              | `2`            |
| Volume   | `top_solid_layers`                   | integer              | `0`            |
| Volume   | `bottom_solid_layers`                | integer              | `0`            |
| Volume   | `external_perimeter_extrusion_width` | number or percentage | `0.7`          |

Available settings depend on printer technology, selected profiles, and the current PrusaSlicer build.

## `HwPrinterConfig`

`HwPrinterConfig` has three read-only properties:

```lua
printer.name -> string
printer.tool_count -> integer
printer.tools -> HwToolConfig[]
```

`tools` is a Lua-facing array indexed from `1`.

```lua
local printer = api.project:current_bed():printer_config()

print("Printer: " .. printer.name)
print("Tools: " .. printer.tool_count)

for index = 1, printer.tool_count do
    local diameter = printer.tools[index]:nozzle_diameter()
    print("Tool " .. index .. " nozzle: " .. tostring(diameter))
end
```

## `HwToolConfig`

### `HwToolConfig:feature`

```lua
tool:feature(name) -> nil|boolean|number|string|table
```

Returns a hardware feature by name.
Missing features and JSON null values return `nil`.

Feature values are converted recursively:

- JSON booleans, numbers, and strings become the matching Lua values.
- JSON arrays become 1-based Lua arrays.
- JSON objects become string-keyed Lua tables.

```lua
local tool = api.project:current_bed():printer_config().tools[1]
local diameter = tool:feature("nozzle_diameter")

if diameter ~= nil then
    print("Nozzle diameter: " .. diameter)
end
```

Hardware feature names are profile data and can vary.
Check for `nil` instead of assuming a feature exists.

### `HwToolConfig:nozzle_diameter`

```lua
tool:nozzle_diameter() -> number?
```

Convenience method for the numeric `nozzle_diameter` feature.
Returns `nil` when the feature is missing.

```lua
local printer = api.project:current_bed():printer_config()
local nozzle = printer.tools[1]:nozzle_diameter() or 0.4
```

[Back to the API index](index.md)
