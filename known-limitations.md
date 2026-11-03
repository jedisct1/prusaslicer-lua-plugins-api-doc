# Current limitations

This page describes behavior confirmed in PrusaSlicer `3.0.0-alpha13` at commit `ff3d6677cf`.
These are current implementation limits, not recommended API design.

## Primitive mesh bounds are stale

Primitive constructors create mesh geometry without updating the mesh's cached bounding box.
`mesh:bounds()` can therefore report all zeroes for a mesh returned by functions such as `api.make_cube` or `api.make_cylinder`.

`Mesh:translate` changes vertices but does not refresh that cache either.

Do not use primitive `Mesh:bounds()` values for layout.
Keep the dimensions you passed to the constructor, or use bounds from a successfully loaded or generated asset whose mesh statistics are populated.

## API version requirements are not enforced

PrusaSlicer enforces `min_slicer_version` and `max_slicer_version` during bundle discovery.
It parses `required_apis` names and versions, but does not compare the required API versions with the implementation.
Unknown API names and malformed versions still make a manifest invalid.

## Some configuration values are opaque

`ConfigBox:value` directly returns booleans, integers, optional integers, ordinary numbers, and strings.
It can also return 2D vector, percentage, and float-or-percentage values.

The project API does not register public Lua fields or conversion methods for the 2D vector and percentage types.
Those two types remain opaque.
Float-or-percentage values support `is_percentage` and `get_abs_value(ratio_over)`.

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

## Top-level runtime errors may not stop a command

During discovery, PrusaSlicer logs a top-level runtime error and then still checks the globals created before that error.
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
For local ZIP installation, PrusaSlicer has no UI or CLI command that imports that public key.
The developer or tester must place it there manually.
Marketplace installation stores a versioned bundle key automatically and uses a separate key lookup.

## Installation replacement is not transactional

When installing a signed bundle whose ID already exists, PrusaSlicer removes the existing bundle directory before extracting the new one.
If extraction then fails, the previous installed copy is not restored automatically.

Keep the source bundle and release ZIP somewhere outside the user plugin directory.

## No management or background APIs

There is no plugin uninstall CLI command, general enable or disable switch, dependency system, background task API, event callbacks, custom UI API, or persistent plugin storage.
The Plugin Marketplace can install updates and uninstall user bundles.
Application-bundled plugins are protected from removal.

Remove a development bundle manually, then choose **Rescan Plugins**.
