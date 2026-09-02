# Examples

The Lua listings are complete command files unless a section shows several files.
Asset examples also need the files shown in their bundle layouts.
Put them in a bundle with a valid [manifest](bundles-and-manifests.md).

## Recipes

- [Examples](#examples)
  - [Recipes](#recipes)
  - [Parameterized box](#parameterized-box)
  - [Plate with a hole and modifier](#plate-with-a-hole-and-modifier)
  - [Load an STL and add an engraved label](#load-an-stl-and-add-an-engraved-label)
  - [Reusable helper module](#reusable-helper-module)
  - [Read printer hardware](#read-printer-hardware)
  - [Change selected presets](#change-selected-presets)
  - [Add temperature changes by height](#add-temperature-changes-by-height)
  - [SVG logo as a support blocker](#svg-logo-as-a-support-blocker)

## Parameterized box

This example uses all four parameter declarations and adds one object.

PrusaSlicer 3.0.0-alpha11 has swapped `int` and `float` controls.
The declarations below describe the intended API behavior.

```lua
info = {
    id = "box",
    type = "project.plugin",
    title = "Add a box",
    menu = "Examples/Add a box",
    params = {
        {name = "name", label = "Name", type = "string", default = "Box"},
        {name = "width", label = "Width", type = "float", default = 30},
        {name = "height", label = "Height", type = "float", default = 20},
        {name = "copies", label = "Copies", type = "int", default = 1},
        {name = "dense", label = "Use dense infill", type = "bool", default = false}
    }
}

function execute(opts)
    assert(opts.width > 0, "Width must be positive")
    assert(opts.height > 0, "Height must be positive")
    assert(opts.copies >= 1, "Copies must be at least 1")

    local extra_copies = {}

    for index = 2, math.floor(opts.copies) do
        extra_copies[#extra_copies + 1] = {
            mesh = api.make_cube(opts.width, opts.height, 5),
            type = VolumeType.Solid,
            translate = {x = (index - 1) * (opts.width + 5)},
        }
    end

    api.project:add_object {
        mesh = api.make_cube(opts.width, opts.height, 5),
        other_volumes = extra_copies,
        object_params = {
            fill_density = opts.dense and "60%" or "15%"
        }
    }

    print("Added " .. opts.name .. " with " .. math.floor(opts.copies) .. " solid parts")
end
```

The copies are separate solid parts of one project object.
Separate `add_object` calls would each be centered on the selected bed and would overlap.

## Plate with a hole and modifier

This example shows explicit volume roles and local transforms.

```lua
info = {
    id = "modified_plate",
    type = "project.plugin",
    title = "Add a modified plate",
    menu = "Examples/Modified plate"
}

function execute(opts)
    local plate = api.make_cube(60, 40, 4)
    local hole = api.make_cylinder(5, 4, 8)
    local dense_region = api.make_cube(20, 40, 4)

    api.project:add_object {
        mesh = plate,
        other_volumes = {
            {
                mesh = hole,
                type = VolumeType.Negative,
                translate = {x = 30, y = 20}
            },
            {
                mesh = dense_region,
                type = VolumeType.Modifier,
                translate = {x = 40},
                params = {
                    fill_density = "80%",
                    perimeters = 4
                }
            }
        }
    }
end
```

Without the explicit `type`, the dense region would still become a modifier because it has `params`.
The explicit name makes the intent easier to read and protects the code if that default changes.

## Load an STL and add an engraved label

Bundle layout:

```text
com.example.labeled-part/
  manifest.json
  labeled_part.lua
  assets/
    bracket.stl
```

```lua
info = {
    id = "labeled_part",
    type = "project.plugin",
    title = "Add labeled bracket",
    menu = "Examples/Labeled bracket",
    params = {
        {name = "label", label = "Label", type = "string", default = "LEFT"}
    }
}

function execute(opts)
    local bracket = api.load_stl("assets/bracket.stl")
    local label = api.emboss_text {
        font = api.get_default_font(),
        text = opts.label,
        line_height = 6
    }

    api.project:add_object {
        mesh = bracket,
        other_volumes = {
            {
                mesh = label,
                type = VolumeType.Negative,
                rotate = {x = 90},
                translate = {x = 5, y = 1, z = 8}
            }
        }
    }
end
```

Text depth is fixed at 1 mm in alpha11.
Adjust the translation so the negative volume intersects the model surface.

## Reusable helper module

Bundle layout:

```text
com.example.badges/
  manifest.json
  add_badge.lua
  badge.lua
```

`badge.lua`:

```lua
local M = {}

function M.make(text, width, height)
    local base = api.make_cube(width, height, 2)
    local letters = api.emboss_text {
        font = api.get_default_font(),
        text = text,
        line_height = height * 0.45
    }

    return base, {
        {
            mesh = letters,
            type = VolumeType.Solid,
            translate = {x = 3, y = height * 0.3, z = 2}
        }
    }
end

return M
```

`add_badge.lua`:

```lua
info = {
    id = "badge",
    type = "project.plugin",
    title = "Add a badge",
    menu = "Examples/Badge",
    params = {
        {name = "text", label = "Text", type = "string", default = "HELLO"}
    }
}

function execute(opts)
    local badge = require("badge")
    local base, volumes = badge.make(opts.text, 50, 20)

    api.project:add_object {
        mesh = base,
        other_volumes = volumes
    }
end
```

`require` stays inside `execute` because it is unavailable during command discovery.

## Read printer hardware

This example reads hardware safely and chooses a size based on the first nozzle.

```lua
info = {
    id = "nozzle_marker",
    type = "project.plugin",
    title = "Add nozzle marker",
    menu = "Examples/Nozzle marker"
}

function execute(opts)
    local bed = api.project:current_bed()
    local printer = bed:printer_config()
    local nozzle = printer.tools[1]:nozzle_diameter() or 0.4

    print("Printer: " .. printer.name)
    print("First nozzle: " .. nozzle)

    api.project:add_object {
        mesh = api.make_cylinder(nozzle * 5, 2, 10)
    }
end
```

`printer.tools` is 1-based.
Preset methods that take a tool or material index are 0-based.

## Change selected presets

This example changes settings that the current Lua setter supports.

```lua
info = {
    id = "draft_settings",
    type = "project.plugin",
    title = "Use draft settings",
    menu = "Examples/Use draft settings"
}

function execute(opts)
    local bed = api.project:current_bed()
    local print_settings = bed:print_presets()

    print_settings:set("layer_height", 0.3)
    print_settings:set("fill_density", "10%")
    print_settings:set("brim_type", "outer_only")
    print_settings:set("brim_width", 4)

    local actual_height = print_settings:value("layer_height")
    assert(math.abs(actual_height - 0.3) < 1e-9, "Could not set layer height")

    api.project:add_object {
        mesh = api.make_cube(20, 20, 10)
    }
end
```

`ConfigBox:set` returns no failure status for an unknown or unsupported setting.
An unsupported override can still be enabled while keeping its previous stored value.
Export the configuration schema and test each key you plan to use.

## Add temperature changes by height

This simplified tower adds custom G-code entries in ascending Z order.

```lua
info = {
    id = "temperature_steps",
    type = "project.plugin",
    title = "Add temperature steps",
    menu = "Examples/Temperature steps",
    params = {
        {name = "start", label = "Start temperature", type = "int", default = 230},
        {name = "step", label = "Temperature drop", type = "int", default = 5},
        {name = "count", label = "Step count", type = "int", default = 4}
    }
}

function execute(opts)
    local bed = api.project:current_bed()
    local count = math.floor(opts.count)
    local section_height = 10

    assert(count >= 1, "Step count must be positive")

    api.project:clear_layer_custom_steps(bed)

    for index = 1, count do
        local z = (index - 1) * section_height + 0.2
        local temperature = math.floor(opts.start - (index - 1) * opts.step)
        api.project:insert_layer_custom_gcode(bed, z, "M104 S" .. temperature)
    end

    api.project:add_object {
        mesh = api.make_cube(20, 20, count * section_height)
    }
end
```

The call to `clear_layer_custom_steps` removes all earlier custom G-code on that bed.
A production plugin should make that effect clear to the user.

## SVG logo as a support blocker

```lua
info = {
    id = "logo_blocker",
    type = "project.plugin",
    title = "Add logo support blocker",
    menu = "Examples/Logo support blocker"
}

function execute(opts)
    local model = api.load_stl("assets/model.stl")
    local logo = api.emboss_svg("assets/logo.svg", 10)

    api.project:add_object {
        mesh = model,
        other_volumes = {
            {
                mesh = logo,
                type = VolumeType.SupportBlocker,
                translate = {x = 10, y = 10}
            }
        }
    }
end
```

Both asset paths are relative to this command file and must remain inside its sandbox.
