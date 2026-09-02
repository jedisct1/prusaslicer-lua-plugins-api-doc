# Current limitations

This page describes behavior confirmed in PrusaSlicer `3.0.0-alpha11` at commit `6f510128d7`.
These are current implementation limits, not recommended API design.

## Numeric parameters are swapped

The `float` and `int` dialog controls are connected to the wrong validators.

- `type = "float"` rounds the entered value to a whole number and passes a Lua integer.
- `type = "int"` accepts a decimal and passes a Lua float.

For example, entering `1.5` in a `float` field produces `2`.
Entering `2.5` in an `int` field produces `2.5`.

This is tracked upstream as [PrusaSlicer issue #15611](https://github.com/prusa3d/PrusaSlicer/issues/15611) and has been acknowledged as internal issue SPE-4074.

Do not silently swap the declared types in a published plugin.
That would break its meaning after PrusaSlicer fixes the bug.
For alpha11 testing, validate values inside `execute` and explain the restriction to testers.

## `emboss_text` ignores `depth`

`api.emboss_text` accepts a `depth` field, but alpha11 does not read it.
Generated text is always 1 mm deep.

This input:

```lua
local text = api.emboss_text {
    font = api.get_default_font(),
    text = "Hello",
    depth = 3
}
```

still produces 1 mm depth.
The `line_height` and `per_glyph` fields do work.

## Primitive mesh bounds are stale

Primitive constructors create mesh geometry without updating the mesh's cached bounding box.
In alpha11, `mesh:bounds()` can therefore report all zeroes for a mesh returned by functions such as `api.make_cube` or `api.make_cylinder`.

`Mesh:translate` changes vertices but does not refresh that cache either.

Do not use primitive `Mesh:bounds()` values for layout in this version.
Keep the dimensions you passed to the constructor, or use bounds from a successfully loaded or generated asset whose mesh statistics are populated.

## Compatibility fields are not enforced

PrusaSlicer parses these manifest values but does not currently compare them with the running application or API:

- `min_slicer_version`
- `max_slicer_version`
- Values inside `required_apis`

Unknown API names and malformed versions still make a manifest invalid.
Valid but incompatible version ranges do not stop loading in alpha11.

## Some configuration values are opaque

`ConfigBox:value` directly returns booleans, integers, optional integers, ordinary numbers, and strings.
It can also return 2D vector, percentage, and float-or-percentage values.

Alpha11 does not register public Lua fields or conversion methods for those three complex types.
Lua receives them as opaque values with no useful documented fields or conversions.

There is also no API to list a ConfigBox's keys or learn their types.

## Configuration write failures are hard to detect

`ConfigBox:set`, object `object_params`, and volume `params` use the same narrow setter.
It supports integer, number, percentage, float-or-percentage, and enum destinations.

Boolean, ordinary string, vector, and other destination types are not supported.
An unknown key is not assigned.
A known unsupported destination type is also not assigned, and an invalid enum string is logged but not assigned.

For an override setting, a known unsupported type or invalid enum can still enable that override while leaving its stored value unchanged.
`ConfigBox:set` returns no status.
The object and volume parameter paths log the same misleading `not found` message for unknown and known unsupported settings.

Read a setting back when its return type is usable and the result matters.

## Several commands without a menu can crash registration

The `menu` field is optional in the metadata parser.
When two or more commands omit it, alpha11 groups their empty menu paths as a conflict and then tries to edit the last part of an empty path.
This is an invalid memory access and can crash PrusaSlicer.

Give every command a non-empty `menu` value in this release.

## Top-level runtime errors may not stop a command

During discovery, alpha11 logs a top-level runtime error and then still checks the globals created before that error.
If valid `info` and `execute` values already exist, the command can still appear in the menu.
During a run, it can also continue to look up and call `execute` after a top-level error.

Keep the top level limited to declarations and simple constants.
Put work inside `execute`, validate early, and remember that a failed execution does not roll back project changes automatically.

## The main volume ignores `type`

In `api.project:add_object`, a `type` field on the top-level object definition is ignored.
The main mesh is created as a solid model part.

`type` works for entries in `other_volumes`.

## `get_font` has no not-found result

`api.get_font(name)` does a case-sensitive substring search.
When it finds no match, it returns the default font instead of `nil` or an error.

Compare the returned font's `name` when an exact font matters.

## Installing an author key is manual

The GUI ZIP installer requires an author public key in `authorized_authors/<author>.pem`.
Alpha11 has no UI or CLI command that imports that public key.
The developer or tester must place it there manually.

## Installation replacement is not transactional

When installing a signed bundle whose ID already exists, PrusaSlicer removes the existing bundle directory before extracting the new one.
If extraction then fails, the previous installed copy is not restored automatically.

Keep the source bundle and release ZIP somewhere outside the user plugin directory.

## No management or background APIs

Alpha11 has no plugin uninstall command, enable or disable switch, dependency system, automatic updater, background task API, event callbacks, custom UI API, or persistent plugin storage.

Remove a development bundle manually, then choose **Rescan Plugins**.
