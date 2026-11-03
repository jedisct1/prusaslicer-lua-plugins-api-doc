# Commands and parameters

Each plugin entry is a `.lua` file with a global `info` table and a global `execute(opts)` function.
Use `project.plugin` for a command that changes the project and `print.print_mode` for a print mode that returns configuration overrides.

## The `info` table

```lua
info = {
    id = "add_label",
    type = "project.plugin",
    title = "Add a label",
    menu = "Model helpers/Add a label",
    params = {
        {
            name = "text",
            label = "Label text",
            type = "string",
            default = "Hello"
        },
        {
            name = "raised",
            label = "Raised text",
            type = "bool",
            default = true
        }
    }
}
```

| Field       | Required | Meaning                                                                                        |
| ----------- | -------- | ---------------------------------------------------------------------------------------------- |
| `id`        | Yes      | Local command ID. The bundle ID is added in front of it.                                       |
| `type`      | Yes      | `project.plugin` or `print.print_mode`.                                                        |
| `title`     | No       | Display title. The full command ID is used when this is missing.                               |
| `menu`      | No       | Plugins menu path for a project command, split on `/`. Print modes do not appear in this menu. |
| `params`    | No       | Array of parameter definitions.                                                                |
| `condition` | No       | Print mode compatibility expression as a string. See [Print modes](#print-modes).              |

The full ID is `<bundle id>.<local command id>`.
For example:

```text
com.example.model-tools.add_label
```

Keep the local ID short and unique inside its bundle.

## Menu paths

This value:

```lua
menu = "Calibration/Temperature tower"
```

creates this path:

```text
Plugins > Calibration > Temperature tower
```

When a project command has no `menu`, PrusaSlicer uses its full command ID as the menu label.

Avoid leading, trailing, or repeated `/` characters.
The parser splits the string literally and does not clean up empty path parts.

If commands register the same menu path, PrusaSlicer adds a suffix to the last menu label.
It normally uses the bundle ID when the conflicting commands come from different bundles.

## Parameter definitions

Each item in `params` has this shape:

```lua
{
    name = "height",
    label = "Height in mm",
    type = "float",
    default = 10
}
```

| Field            | Required       | Meaning                                                                          |
| ---------------- | -------------- | -------------------------------------------------------------------------------- |
| `name`           | Yes            | Unique key placed in the `opts` table. Names must not repeat within one command. |
| `label`          | No             | Text shown beside the control. Defaults to `name`.                               |
| `type`           | Yes            | `string`, `float`, `int`, `bool`, `combobox`, or `font`.                         |
| `default`        | No             | Initial value with the matching Lua type.                                        |
| `allowed_values` | For `combobox` | Non-empty array of strings offered by the control.                               |

For project commands, omitted defaults give an empty string, `false`, or zero for scalar controls.
Provide explicit defaults when the initial value matters.

### String

```lua
{name = "note", label = "Note", type = "string", default = "Test 1"}
```

The value passed to `execute` is a Lua string.

### Boolean

```lua
{name = "add_brim", label = "Add brim", type = "bool", default = true}
```

The value passed to `execute` is a Lua boolean.

### Integer and float

```lua
{name = "copies", label = "Copies", type = "int", default = 2}
{name = "height", label = "Height", type = "float", default = 12.5}
```

`int` uses an integer control and returns an integer.
`float` uses a decimal control and returns a number.

### Combo box

```lua
{
    name = "shape",
    label = "Shape",
    type = "combobox",
    allowed_values = {"Cube", "Cylinder"},
    default = "Cube"
}
```

The value passed to `execute` is the selected string, not an index.
Use the exact type name `combobox`.
Always supply at least one allowed value and choose a default from that list.

### Font chooser

```lua
{name = "font", label = "Font", type = "font"}
```

For a project command, this control lists the fonts returned by PrusaSlicer's font manager.
The selected value in `opts.font` is a `FontDescriptor` that can be passed directly to `api.emboss_text`.
An optional string `default` can match a font name or font path.
Without one, the dialog uses the application's default font.
Font parameters are not supported by the print-mode parameter storage.

Any parameter type other than the six listed above is unsupported and may make the dialog fail instead of showing a useful plugin error.

## Project command `execute(opts)`

PrusaSlicer calls this function after the user clicks Run:

```lua
function execute(opts)
    print("Adding " .. opts.copies .. " copies")

    local copies = {}

    for index = 2, math.floor(opts.copies) do
        copies[#copies + 1] = {
            mesh = api.make_cube(10, 10, 10),
            type = VolumeType.Solid,
            translate = {x = (index - 1) * 15}
        }
    end

    api.project:add_object {
        mesh = api.make_cube(10, 10, 10),
        other_volumes = copies
    }
end
```

`opts` contains one value for each declared parameter.
When there are no parameters, it is an empty table.
The function's return value is ignored.

The dialog appears even if the command has no parameters.
Run calls `execute`; Cancel closes the dialog without running it.

The last submitted values are remembered only for the most recently run command.
They are kept in memory, cleared by a plugin rescan, and are not a persistent storage API.

## Print modes

A print mode uses `type = "print.print_mode"` and belongs to a bundle that declares `"print.print_mode": "1.0.0"` in `required_apis`.
It is evaluated as part of preset configuration rather than from the Plugins menu.
Its `execute(opts)` must return a table of overrides:

```lua
info = {
    id = "sparse_infill",
    type = "print.print_mode",
    title = "Sparse infill",
    params = {
        {name = "density", label = "Infill percentage", type = "float", default = 10.0}
    }
}

function execute(opts)
    return {fill_density = opts.density}
end
```

Use explicit scalar defaults for print-mode parameters.
The evaluator uses the default value's type to accept supplied parameter values, with integer-to-float conversion allowed.
An omitted `float` default falls back to an integer zero, so use a decimal default such as `10.0`.

The print-mode runtime provides `api.config:value(key)` to read the input configuration.
An unknown key raises an error.
Scalar values use Lua booleans, numbers, strings, or `{x = ..., y = ...}` points; vector values become 1-based tables.
Percentages and float-or-percentage inputs become numbers.
The project geometry and preset mutation APIs are unavailable in this runtime.

Return scalar values for ordinary override keys.
For a per-slot key, a scalar applies to every material slot; a 1-based table applies only to the slots it names.
For example, `filament_soluble = {[2] = true}` changes only slot 2.
Return percentage overrides as numbers such as `fill_density = 10`, rather than strings such as `"10%"`.
Only macro-overridable keys and supported scalar types are accepted.
Unknown or non-macro keys, tables for ordinary keys, and unsupported configuration value types are logged and skipped.
A value that cannot be converted to the expected scalar type can instead fail evaluation.
If execution raises an error or returns something other than a table, the mode reports an error and its overrides are cleared.

An optional `condition` uses the preset expression language, not Lua:

```lua
condition = "printer.tool_count > 1 or feeder.base_model =~ /.*MMU.*/"
```

It can read `printer.*`, `feeder.*`, and `sheet.*` variables.
Without a condition, a mode is compatible with every printer.
A condition that fails evaluation or does not return a boolean makes the mode incompatible.
The project-command runner does not enforce this condition.

## Several commands in one bundle

Put each command in its own entry file:

```text
com.example.shapes/
  manifest.json
  add_cube.lua
  add_cylinder.lua
  geometry_helpers.lua
```

`add_cube.lua` and `add_cylinder.lua` can each define `info` and `execute`.
`geometry_helpers.lua` can return a normal module table without defining them.

Every `.lua` file is evaluated during scanning.
Helper modules without command metadata are skipped with an informational log message.
This is expected.

For project commands, do not call `require` at the top of an entry file.
Call it inside `execute` so it runs only after the restricted module loader has been installed:

```lua
function execute(opts)
    local helpers = require("geometry_helpers")
    helpers.add_shape(opts)
end
```

Read [Runtime and sandbox](runtime-and-sandbox.md) for the complete lifecycle and module rules.
