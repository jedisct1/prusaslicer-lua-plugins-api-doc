# Commands and parameters

Each command is a `.lua` file with a global `info` table and a global `execute(opts)` function.

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

| Field    |       Required | Meaning                                                                                                                               |
| -------- | -------------: | ------------------------------------------------------------------------------------------------------------------------------------- |
| `id`     |            Yes | Local command ID. The bundle ID is added in front of it.                                                                              |
| `type`   |            Yes | Must be exactly `project.plugin`.                                                                                                     |
| `title`  |             No | Dialog title. The full command ID is used when this is missing.                                                                       |
| `menu`   | Yes in alpha11 | Plugins menu path, split on `/`. The parser accepts a missing value, but this release has a crash risk when several commands omit it. |
| `params` |             No | Array of parameter definitions shown in the Run dialog.                                                                               |

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

When one command has no `menu`, PrusaSlicer normally uses the full command ID as its menu label.
Always provide a non-empty `menu` in alpha11.
Two or more commands without one can reach a broken conflict-handling path during registration.
See [Several commands without a menu can crash registration](known-limitations.md#several-commands-without-a-menu-can-crash-registration).

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

| Field     |        Required | Meaning                                                                          |
| --------- | --------------: | -------------------------------------------------------------------------------- |
| `name`    |             Yes | Unique key placed in the `opts` table. Names must not repeat within one command. |
| `label`   |              No | Text shown beside the control. Defaults to `name`.                               |
| `type`    |             Yes | One of `string`, `float`, `int`, or `bool`.                                      |
| `default` | Yes in practice | Initial value. Always provide the matching Lua type.                             |

The current parser still expects this field.
Always include it.

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

The intended behavior is an integer control for `int` and a decimal control for `float`.
PrusaSlicer 3.0.0-alpha11 wires these two controls in reverse.
At runtime, `float` rounds to an integer and `int` accepts and returns a decimal value.
See [Numeric parameters are swapped](known-limitations.md#numeric-parameters-are-swapped).

Any other type is unsupported and may make the dialog fail instead of showing a useful plugin error.

## `execute(opts)`

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

Do not call `require` at the top of an entry file.
Call it inside `execute` so it runs only after the restricted module loader has been installed:

```lua
function execute(opts)
    local helpers = require("geometry_helpers")
    helpers.add_shape(opts)
end
```

Read [Runtime and sandbox](runtime-and-sandbox.md) for the complete lifecycle and module rules.
