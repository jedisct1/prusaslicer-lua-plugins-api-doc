# Runtime and sandbox

PrusaSlicer plugins run as short, user-triggered commands.
Understanding the two-stage lifecycle prevents most loading problems.

## The lifecycle

### 1. Discovery

PrusaSlicer scans its built-in plugin directory first, then the user plugin directory.
For each bundle, it recursively finds every filename ending in `.lua`.

Each file is evaluated in a new, restricted Lua state.
PrusaSlicer then checks whether the state has a valid global `info` table and global `execute` function.

The discovery state does not contain `api` or the custom `require` function.
This means top-level API work fails during scanning.

Use top-level code only for declarations and calculations that need the basic Lua libraries:

```lua
info = {
    id = "example",
    type = "project.plugin",
    menu = "Examples/Discovery-safe example"
}

local default_radius = 10

local function make_model(radius)
    return api.make_sphere(radius)
end

function execute(opts)
    api.project:add_object {
        mesh = make_model(opts.radius or default_radius)
    }
end
```

The reference to `api` inside `make_model` is safe because the function is not called during discovery.

### 2. User starts a command

The user picks the command from the Plugins menu.
PrusaSlicer opens its parameter dialog.
Cancel stops here.

### 3. Execution

After the user clicks Run, PrusaSlicer:

1. Creates a fresh Lua state.
2. Registers the project API and restricted `require`.
3. Evaluates the chosen entry file again.
4. Builds an `opts` table from the dialog.
5. Calls `execute(opts)`.
6. Requests a scene render.
7. Creates an Execute Plugin undo snapshot.
8. Destroys the Lua state.

Globals and loaded modules do not survive the run.
Every invocation starts fresh.

There are no other Lua callbacks.
In particular, there is no load, unload, project-open, project-save, selection-change, slicing, timer, or shutdown callback.

## Lua version and available libraries

Official dependency builds use Lua 5.4.8.
A custom system build may use another compatible Lua version selected at build time.

The runtime opens only these standard libraries:

- Base
- `table`
- `math`
- `string`

Useful base functions such as `assert`, `error`, `ipairs`, `pairs`, `pcall`, `print`, `tonumber`, `tostring`, and `type` are available.

These libraries are not opened:

- `io`
- `os`
- `package`
- `debug`
- `coroutine`
- `utf8`

The base functions `dofile`, `loadfile`, and `load` are removed.

`getmetatable` and `setmetatable` work only when their target is a Lua table.
Using them on a string, number, function, or API userdata raises `Invalid metadata target`.

## Global values

During execution, PrusaSlicer provides:

- `api`, the project plugin API table.
- `api.project`, an object bound to the selected project.
- `VolumeType`, the volume role constants.
- The restricted `require` function.
- The `BoundingBox`, `Mesh`, `ModelElement`, `ConfigBox`, `HwToolConfig`, `HwPrinterConfig`, `BedInstRef`, `Font`, and `ProjectApi` type tables.

The API types have no public constructors.
You get their values from API functions.

`api` is already global.
This is correct:

```lua
local cube = api.make_cube(10, 10, 10)
```

This tries to load a local file named `api.lua` and is normally wrong:

```lua
local api = require("api")
```

## Restricted `require`

Use `require` to load another Lua file from the entry script's directory:

```text
com.example.plugin/
  main.lua
  helpers.lua
  geometry/
    badge.lua
```

```lua
function execute(opts)
    local helpers = require("helpers")
    local badge = require("geometry/badge")

    helpers.run(opts, badge)
end
```

The loader follows these rules:

- It appends `.lua` to the module name.
- It does not convert dots to directory separators.
- It resolves the file relative to the entry script's directory.
- It rejects a path that resolves outside that directory.
- It caches the returned value for the current invocation.
- If the module returns nothing, `require` returns and caches `true`.

For example:

| Call                        | File requested       |
| --------------------------- | -------------------- |
| `require("helpers")`        | `helpers.lua`        |
| `require("geometry/badge")` | `geometry/badge.lua` |
| `require("geometry.badge")` | `geometry.badge.lua` |

There is no `package.path` or `package.loaded`.
The cache is private to the custom loader and disappears after the command.

Do not rely on standard Lua circular-module handling.
The custom loader caches a module only after it finishes running, so circular `require` calls can recurse until they fail.

## Helper module pattern

Every helper `.lua` file is also evaluated by the discovery scan.
Keep its top level free of API calls and asset loading:

```lua
local M = {}

function M.add_cube(size)
    api.project:add_object {
        mesh = api.make_cube(size, size, size)
    }
end

return M
```

Return one table from each helper module.
The custom loader caches one returned value and does not provide the extra metadata behavior of Lua's standard package loader.

It is fine for a declared function to use `api` later.
The important part is that the module does not call that function at top level.

PrusaSlicer logs an informational message when a helper file has no plugin metadata.
That message is expected and does not mean the module is broken.

## Asset paths

`require`, `api.load_stl`, and `api.emboss_svg` use the same path sandbox.
Paths are relative to the command entry file:

```lua
function execute(opts)
    local model = api.load_stl("assets/model.stl")
    local logo = api.emboss_svg("assets/logo.svg", 1.2)
end
```

Parent traversal, absolute paths outside the entry directory, and symlinks that resolve outside it are rejected.

If an entry file is nested, its sandbox starts at that nested directory.
For example, `commands/main.lua` cannot load `../assets/model.stl` even if both paths belong to the same bundle.
Keeping entry files at the bundle root avoids this surprise.

The sandbox provides read access only through the exposed asset functions and module loader.
There is no general file-writing API.

## Errors and output

`print` writes to the process's normal output, and there is no in-app plugin console.
Start PrusaSlicer from a terminal to see printed diagnostics reliably.
Runtime failures go through PrusaSlicer's application log.

Syntax and metadata errors during scanning are logged and normally keep the affected command out of the menu.
A top-level runtime error is different.
If the file defines valid `info` and `execute` globals before the failing line, alpha11 may still register it.

During a command run, alpha11 also continues to look up and call `execute` after a top-level runtime error.
An error raised by `execute` itself is logged rather than shown in the parameter dialog.
Keep entry files declarative at the top level and put working code inside `execute`.

Use clear assertions around values that matter:

```lua
function execute(opts)
    assert(opts.size > 0, "Size must be greater than zero")

    api.project:add_object {
        mesh = api.make_cube(opts.size, opts.size, opts.size)
    }
end
```

PrusaSlicer takes an undo snapshot after the run attempt, including after a caught execution error.
Execution is not transactional.
If a command changes the project and then fails, the earlier changes remain until the user undoes them.
Validate inputs and prepare fallible data before making the first project change.

## Permissions and persistent state

There is no permissions field in `manifest.json` and no permission prompt.
The available API defines the boundary:

- A plugin can change the selected project and supported preset values.
- It can read Lua modules, STL files, and SVG files within its entry directory.
- It cannot use a provided network, process, general filesystem, or application settings API.

There is no plugin storage API.
Global variables disappear after each invocation.
The dialog remembers the last submitted values only for the most recently run command, only until a rescan or application restart.
