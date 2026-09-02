# PrusaSlicer 3 Lua plugin API

This guide explains how to build Lua plugins for PrusaSlicer 3.
It is written for plugin developers, so you do not need to know the PrusaSlicer C++ codebase.

The API currently lets a plugin add geometry to the active project, change some print settings, read printer details, and add custom G-code at chosen layer heights.
A plugin runs when the user picks its command from the Plugins menu.

## Version covered

This reference was checked against PrusaSlicer `3.0.0-alpha11`, tag `version_3.0.0-alpha11`, commit `6f510128d7`.
The plugin API is new and still changing.
The [current limitations](known-limitations.md) page lists behavior that is especially likely to change.

## Start here

| If you want to...                      | Read...                                                     |
| -------------------------------------- | ----------------------------------------------------------- |
| Make your first plugin                 | [Getting started](getting-started.md)                       |
| Understand `manifest.json`             | [Bundles and manifests](bundles-and-manifests.md)           |
| Add a Plugins menu command             | [Commands and parameters](commands-and-parameters.md)       |
| Understand when and how Lua runs       | [Runtime and sandbox](runtime-and-sandbox.md)               |
| Look up a function or type             | [API reference](api/index.md)                               |
| Copy practical patterns                | [Examples](examples.md)                                     |
| Create keys, sign, and install a ZIP   | [Packaging and installation](packaging-and-installation.md) |
| Fix a plugin that does not load or run | [Troubleshooting](troubleshooting.md)                       |

## What a plugin can do

A project plugin can:

- Create built-in shapes such as cubes, cylinders, spheres, and toruses.
- Load STL files stored inside the command entry script's directory.
- Turn SVG artwork or installed fonts into a mesh.
- Add objects with solid, negative, modifier, support blocker, or support enforcer volumes.
- Read and change supported print, printer, tool, material, object, and volume settings.
- Read the current printer name, tool count, nozzle diameter, and hardware features.
- Add or clear custom G-code entries at layer heights.
- Split code into local Lua modules with a restricted version of `require`.

## What it cannot do

The current API is a small, one-shot project automation API.
It is not a general application extension system.

There are no Lua callbacks for project load, save, slicing, selection changes, timers, or shutdown.
There is no background execution, network API, general file API, persistent storage, custom window API, or plugin-to-plugin dependency system.

## Basic conventions

- Distances are in millimeters unless a function says otherwise.
- Angles passed to the public API are in degrees.
- Lua arrays start at `1`.
- Tool and material indices passed to preset methods start at `0`.
- Methods use `:` and fields use `.`.
- `api` and `VolumeType` are globals provided by PrusaSlicer.
  Do not call `require("api")`.

Here is a small example:

```lua
info = {
    id = "add_cube",
    type = "project.plugin",
    title = "Add a cube",
    menu = "Examples/Add a cube"
}

function execute(opts)
    api.project:add_object {
        mesh = api.make_cube(20, 20, 20)
    }
end
```

See [Getting started](getting-started.md) for the matching manifest and folder layout.
