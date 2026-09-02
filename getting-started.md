# Getting started

This tutorial creates a plugin that adds a 20 mm cube to the active project.

You need PrusaSlicer 3, a text editor, and basic familiarity with Lua tables and functions.
You do not need a compiler, an SDK, or a PrusaSlicer source checkout.

## 1. Open the user plugin folder

In PrusaSlicer, open the Plugins menu and choose **Show User Plugins Folder**.
This is the safest way to find the right folder on every operating system and in development builds.

Each direct child of this folder is one plugin bundle:

```text
lua/
  com.example.first-plugin/
    manifest.json
    add_cube.lua
```

Loose `.lua` files directly inside the `lua` folder are ignored.
The bundle folder must contain `manifest.json` at its top level.

## 2. Create the manifest

A plugin bundle uses two file formats for different jobs.
`manifest.json` describes the whole bundle and uses JSON syntax.
Each command file uses Lua syntax and defines `info` plus `execute`.
One bundle can contain several command files.

Create `com.example.first-plugin/manifest.json`:

```json
{
  "id": "com.example.first-plugin",
  "name": "My first plugin",
  "version": "1.0.0",
  "min_slicer_version": "3.0.0",
  "author": "example-developer",
  "license": "MIT",
  "description": "Small examples for learning the Lua API",
  "required_apis": {
    "project.plugin": "1.0.0"
  }
}
```

The bundle ID should use reverse domain name form so it stays unique.
If you do not own a domain, use a stable namespace that identifies you or your organization.

See [Bundles and manifests](bundles-and-manifests.md) for every field.

## 3. Create the command script

Create `com.example.first-plugin/add_cube.lua`:

```lua
info = {
    id = "add_cube",
    type = "project.plugin",
    title = "Add a cube",
    menu = "Tutorial/Add a cube"
}

function execute(opts)
    local cube = api.make_cube(20, 20, 20)

    api.project:add_object {
        mesh = cube
    }
end
```

The script has two public parts:

- The global `info` table tells PrusaSlicer how to list the command.
- The global `execute(opts)` function does the work after the user clicks Run.

The full command ID is `com.example.first-plugin.add_cube`.
It combines the bundle ID and the script's local ID.

## 4. Rescan and run it

In PrusaSlicer:

1. Choose **Plugins > Rescan Plugins**.
2. Choose **Plugins > Tutorial > Add a cube**.
3. Click **Run**.

PrusaSlicer adds the object to the active project and creates an undo point for the plugin run.

The command always opens a Run dialog, even when it has no parameters.

## 5. Add a parameter

Replace the `info` table and function with this version:

```lua
info = {
    id = "add_cube",
    type = "project.plugin",
    title = "Add a cube",
    menu = "Tutorial/Add a cube",
    params = {
        {
            name = "size",
            label = "Size in mm",
            type = "int",
            default = 20
        }
    }
}

function execute(opts)
    local size = math.floor(opts.size)
    assert(size > 0, "Size must be greater than zero")

    api.project:add_object {
        mesh = api.make_cube(size, size, size)
    }
end
```

Rescan again after changing metadata or adding files.
The dialog now passes a `size` value in `opts`.

PrusaSlicer 3.0.0-alpha11 has a known numeric dialog bug: `int` accepts decimal values and `float` rounds to an integer.
Read [Numeric parameters are swapped](known-limitations.md#numeric-parameters-are-swapped) before designing a parameter form.

## 6. Keep top-level code simple

PrusaSlicer evaluates every `.lua` file while scanning for commands.
It evaluates the chosen entry script again in a fresh Lua state when the command runs.

At the top level, only declare metadata, functions, and simple constants.
Do not use `api`, load assets, or call `require` there.
Put that work inside `execute` or a function called by `execute`.

This is safe:

```lua
local default_size = 20

function execute(opts)
    local helpers = require("helpers")
    helpers.add_cube(opts.size or default_size)
end
```

This is not safe:

```lua
local helpers = require("helpers")
local cube = api.make_cube(20, 20, 20)
```

## Faster setup with the CLI

PrusaSlicer can create the starter bundle for you:

```sh
PrusaSlicer plugin init -d ./plugins com.example.first-plugin
```

The command asks for the display name, author, and license, then creates `hello.lua` and `manifest.json`.
The exact executable name or path depends on your installation.

For day-to-day development, copy or link the resulting bundle directory into the user plugin folder, edit it, and choose **Rescan Plugins**.
Directory-based development does not require a signature.

PrusaSlicer rereads an existing command entry file and its helper modules every time you click Run.
Edits limited to an existing `execute` body or helper module therefore do not strictly require a rescan.
Rescan after changing a manifest or `info`, or after adding or removing files.

## Where to go next

- Read [Commands and parameters](commands-and-parameters.md) to add more menu commands and controls.
- Read [API reference](api/index.md) to add meshes, settings, text, or custom G-code.
- Read [Runtime and sandbox](runtime-and-sandbox.md) before splitting code into modules or loading assets.
- Read [Examples](examples.md) for complete patterns.
