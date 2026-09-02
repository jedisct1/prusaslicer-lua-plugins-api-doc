# Troubleshooting

Most plugin problems come from discovery-time code, exact metadata types, relative paths, or unsupported configuration values.

## My command is not in the Plugins menu

Check these items in order:

1. The bundle is a direct child of the user plugin folder.
2. `manifest.json` is at the bundle root.
3. The manifest is valid JSON and contains every required field.
4. `required_apis` contains only `project.plugin`.
5. The command file ends in `.lua`.
6. The file sets a global `info` table.
7. `info.type` is exactly `project.plugin`.
8. The file defines a global `execute` function.
9. Top-level code does not use `api` or `require`.
10. You chose **Plugins > Rescan Plugins** after editing.

Then inspect the PrusaSlicer log.
Discovery errors are logged, not shown in a plugin dialog.

## A helper module produces a metadata message during rescan

PrusaSlicer evaluates every `.lua` file while looking for commands.
A helper module without `info` is skipped and may produce an informational message such as `Missing info table`.

That is expected if the module loads correctly when its command calls `require`.

## `require` is nil during rescan

The custom loader exists only during actual command execution.
Move `require` inside `execute` or another function called after execution begins.

Wrong:

```lua
local helpers = require("helpers")

function execute(opts)
    helpers.run(opts)
end
```

Right:

```lua
function execute(opts)
    local helpers = require("helpers")
    helpers.run(opts)
end
```

## A module is not found

The custom loader appends `.lua` literally and resolves from the command entry file.

```lua
require("helpers")
```

loads `helpers.lua`.

```lua
require("lib/helpers")
```

loads `lib/helpers.lua`.

```lua
require("lib.helpers")
```

loads `lib.helpers.lua`, not `lib/helpers.lua`.

There is no `package.path`.
Parent traversal and symlink escape are blocked.

## An STL or SVG path is rejected

Paths are relative to the command entry file, not always the bundle root.
They must resolve inside the entry file's directory.

Keep entry scripts at the root or place their assets below their own directory.

`api.load_stl` raises an error for a missing, unreadable, empty, or unsafe file.
Its repair pass may successfully load some damaged STL meshes.
`api.emboss_svg` returns an empty mesh for several read or conversion failures.

## `api` is nil during rescan

The project API exists only in the execution state.
Move the API call inside `execute` or a function that is not called at top level.

Do not use `require("api")`.
That looks for a local `api.lua` file.

## A runtime error does not appear in the dialog

Errors raised by `execute` are written to the PrusaSlicer log.
There is no in-app plugin console in alpha11.

Start the application from a terminal while developing and use `print` for small diagnostics.
Remove temporary diagnostics when the plugin is ready.

Add assertions with useful messages:

```lua
assert(opts.count >= 1, "Count must be at least 1")
```

If the command changed the project before it failed, those earlier changes remain.
Use the Execute Plugin undo entry to restore the previous state.
Validate parameters and load required assets before the first project change when possible.

## A `float` value becomes an integer

Alpha11 connects the `float` parameter to the integer validator and the `int` parameter to the decimal validator.
See [Numeric parameters are swapped](known-limitations.md#numeric-parameters-are-swapped).

This is a known PrusaSlicer defect, not Lua number conversion in your plugin.

## `mesh:bounds()` returns zeroes

Primitive constructors do not update cached mesh statistics in alpha11.
`Mesh:translate` also leaves that cache stale.

Keep known primitive dimensions yourself.
Do not use primitive bounds for placement in this version.

## Text depth stays at 1 mm

Alpha11 accepts but ignores the `depth` field in `api.emboss_text`.
SVG depth works.
See [`emboss_text` ignores `depth`](known-limitations.md#emboss_text-ignores-depth).

## A setting does not change

`ConfigBox:set`, `object_params`, and volume `params` support only a limited set of destination types.
`ConfigBox:set` returns no status for unknown keys, unsupported types, or invalid enum values.
The object and volume parameter paths log a misleading `not found` message for both an unknown key and a known unsupported type.

Check:

- The key is the serialized configuration name, not the UI label.
- The key belongs to the ConfigBox you chose.
- The destination is an integer, number, percentage, number-or-percentage, or enum.
- Enum values use their serialized strings.
- The selected printer technology supports that setting.

Export the current schema:

```sh
PrusaSlicer --export-config-schema config-schema.json
```

Read back ordinary numeric or string values when possible.
For override settings, an unsupported type or invalid enum can enable the override without changing its stored value.

## A preset index fails

Remember the mixed indexing rules:

```lua
printer.tools[1]
bed:tool_print_presets(0)
bed:material_presets(0)
```

The hardware array is 1-based.
Preset method arguments are 0-based.

## A font search picks the wrong font

`api.get_font` uses the first case-sensitive substring match.
If there is no match, it returns the default font.

Inspect `font.name` and use `api.fonts()` when exact selection matters.
Font lists vary by computer.

## A user plugin does not replace a built-in plugin

Built-in bundles are scanned before user bundles.
When two commands have the same full ID, the first registered command wins.

Choose a unique bundle ID and local command ID.

## Menu labels have an unexpected suffix

Two commands registered the same menu path.
PrusaSlicer keeps both and adds a bundle ID or number to the final menu label.

Change one `info.menu` value if the commands should have distinct labels.

## Signed ZIP installation fails

Check all of these:

- `manifest.json` is at archive root.
- `manifest.txt` and `manifest.sign` are present.
- The ZIP was produced after the last file change.
- The manifest author exactly matches the public PEM filename.
- The public key is in `authorized_authors`, not the `lua` directory.
- The public key matches the private key used to sign.
- No file was added, removed, or edited after signing.
- The archive has no unexpected unsigned file.

Use `PrusaSlicer plugin sign` again on a clean bundle rather than editing a signed ZIP.

An installation success message proves that verification and extraction completed.
It does not prove that each Lua command registered.
If the bundle installs but its commands are missing, check the Plugins menu and application log for discovery or duplicate-ID errors.

## Rescan does not preserve my previous parameter values

The dialog remembers values only for the most recently run command and only in memory.
A rescan clears them.
There is no persistent plugin settings API.

## Where to inspect next

- [Runtime and sandbox](runtime-and-sandbox.md) for discovery and execution rules.
- [Commands and parameters](commands-and-parameters.md) for exact metadata shapes.
- [Configuration and hardware](api/configuration-and-hardware.md) for setter limits and indexing.
- [Current limitations](known-limitations.md) for known alpha11 defects.
