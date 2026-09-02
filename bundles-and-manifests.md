# Bundles and manifests

A bundle is a directory that contains one manifest, one or more Lua command files, optional helper modules, and optional assets.

## Recommended layout

```text
com.example.calibration/
  manifest.json
  flow_tower.lua
  temperature_tower.lua
  helpers.lua
  tower.stl
  label.svg
```

One bundle may register several Plugins menu commands.
Every `.lua` file that defines a valid global `info` table and `execute` function becomes a command.
A Lua file without those globals can be used as a helper module.

Keep command entry files at the bundle root when possible.
Relative modules and assets are resolved from the entry file's directory, not from the manifest's directory.

## Complete manifest example

```json
{
  "id": "com.example.calibration",
  "name": "Calibration helpers",
  "version": "1.2.0",
  "min_slicer_version": "3.0.0",
  "max_slicer_version": "3.1.0",
  "author": "example-developer",
  "license": "MIT",
  "description": "Adds small calibration models to the active project",
  "category": "Calibration",
  "web": "https://example.com/calibration-plugin",
  "repo": "https://github.com/example/calibration-plugin",
  "required_apis": {
    "project.plugin": "1.0.0"
  }
}
```

## Fields

| Field                | Required | Type                    | Meaning                                                              |
| -------------------- | -------: | ----------------------- | -------------------------------------------------------------------- |
| `id`                 |      Yes | string                  | Stable ID for the whole bundle.                                      |
| `name`               |      Yes | string                  | Human-readable bundle name.                                          |
| `version`            |      Yes | semantic version string | Version of your bundle.                                              |
| `min_slicer_version` |      Yes | semantic version string | Oldest PrusaSlicer version declared as supported.                    |
| `max_slicer_version` |       No | semantic version string | Newest PrusaSlicer version declared as supported.                    |
| `author`             |      Yes | string                  | Author ID. It also selects the public key used by the ZIP installer. |
| `license`            |      Yes | string                  | License identifier. An SPDX identifier is recommended.               |
| `description`        |       No | string                  | Short description of the bundle.                                     |
| `category`           |       No | string                  | Category metadata. It does not create menu nesting.                  |
| `web`                |       No | string                  | Project website.                                                     |
| `repo`               |       No | string                  | Source repository.                                                   |
| `required_apis`      |      Yes | object                  | Required plugin API names and versions.                              |

All required fields must have exactly the expected JSON type.
Malformed fields can stop the whole bundle from loading.

The bundle name, version, license, description, category, website, and repository are parsed metadata in alpha11.
They are not shown in the Plugins menu or exposed to Lua.
Menu labels and command dialog titles come from each Lua file's `info` table.

Extra fields are currently ignored, but do not use that as a storage mechanism.
They may gain a meaning in a later API version.

## Bundle IDs

Use a stable, unique reverse domain name such as:

```text
com.example.calibration
org.myproject.slicer-tools
```

The `plugin init` and `plugin sign` commands accept letters, digits, `.`, `-`, and `_`.
The initializer's reverse-domain check expects at least three dot-separated parts.
Each part must start with a letter, must not end with `-`, and must be at most 63 characters.
The whole ID must be at most 255 characters.
Values outside these recommendations produce a warning that `--force` can accept.

Do not change an ID just to publish an update.
The GUI installer uses the ID as the destination directory name and replaces the existing directory with that ID.

## Versions

Use semantic versions such as `1.0.0` or `1.2.0-beta.1`.
Invalid version strings make the manifest fail to parse.

In 3.0.0-alpha11, compatibility versions are metadata only.
PrusaSlicer parses `version`, `min_slicer_version`, `max_slicer_version`, and API versions, but it does not compare them with the running version during scan or installation.
Set honest values anyway so your bundle is ready when enforcement is added.

## Required APIs

The only recognized API name is currently `project.plugin`:

```json
"required_apis": {
  "project.plugin": "1.0.0"
}
```

An unknown API name makes the manifest invalid.
The current API version value is parsed but not enforced.

## Author IDs and signing

The author value is more than display text during ZIP installation.
PrusaSlicer looks for this exact file:

```text
<user data directory>/authorized_authors/<author>.pem
```

For example, `"author": "example-developer"` uses `example-developer.pem`.
Author IDs created by the CLI may contain letters, digits, `.`, `-`, and `_`.

You do not need a key while loading an unpacked bundle from the user plugin folder.
You do need one for the signed ZIP installer.
See [Packaging and installation](packaging-and-installation.md).

## How bundles are discovered

PrusaSlicer scans built-in bundles first, then user bundles.
Only direct child directories of those plugin roots are treated as bundles.
A directory counts as a bundle when it has a root-level `manifest.json`.

If two commands have the same full ID, the first one wins.
A user plugin therefore cannot replace a built-in command by reusing its ID.
The built-in root is scanned before the user root, but ordering among duplicate IDs inside the same root depends on directory iteration.
Do not rely on which duplicate user command wins.

Use **Plugins > Rescan Plugins** after changing a manifest or command `info`, adding or removing a file, or removing a bundle.
An existing entry script and its helper modules are reread on every Run, so an edit limited to their execution code does not strictly need a rescan.
