# Fonts, text, and SVG API

[Back to the API index](index.md)

PrusaSlicer can turn installed fonts and local SVG files into meshes.
Those meshes can be used as a main object, a solid volume, or a negative engraving volume.

## `Font`

A `Font` has one read-only property:

```lua
font.name -> string
```

Font availability and names depend on the operating system and installed fonts.
Do not assume that a font installed on your computer exists on another user's computer.

## `api.fonts`

```lua
api.fonts() -> Font[]
```

Returns the available font descriptors as a Lua-facing array indexed from `1`.

```lua
local fonts = api.fonts()

for index = 1, #fonts do
    print(fonts[index].name)
end
```

Numeric indexing and `#fonts` are supported.

## `api.get_default_font`

```lua
api.get_default_font() -> Font
```

Returns the first font in PrusaSlicer's favorite font list, which is also the fallback used by `api.get_font` and `api.emboss_text`.

```lua
local font = api.get_default_font()
print("Using " .. font.name)
```

Use this when consistent fallback behavior matters more than a particular typeface.

## `api.get_font`

```lua
api.get_font(name_substring) -> Font
```

Searches font names for the first case-sensitive substring match.
When no font matches, it returns the default font.
It never returns `nil` for not found.

```lua
local font = api.get_font("Helvetica")

if not string.find(font.name, "Helvetica", 1, true) then
    print("Helvetica was not found, using " .. font.name)
end
```

Use an exact comparison if a substitution would change your model layout.

## `api.emboss_text`

```lua
api.emboss_text(options) -> Mesh
```

Turns text into an extruded mesh.

### Options

```lua
{
    font = Font?,
    text = string,
    line_height = number?,
    per_glyph = boolean?,
    depth = number?
}
```

| Field         | Required | Default      | Meaning                                                           |
| ------------- | -------: | ------------ | ----------------------------------------------------------------- |
| `font`        |       No | Default font | Font descriptor to use.                                           |
| `text`        |      Yes | None         | Text to convert.                                                  |
| `line_height` |       No | 10 mm        | Requested line height.                                            |
| `per_glyph`   |       No | `false`      | Enables per-glyph projection behavior in the text shape provider. |
| `depth`       |       No | 1 mm         | Extrusion depth in millimeters.                                   |

Omitting `font` uses the same font as `api.get_default_font()`.
`depth` controls the mesh extrusion depth.

```lua
local label = api.emboss_text {
    text = "Sample A",
    depth = 1.5,
    line_height = 8,
    per_glyph = false
}
```

Mesh generation failure returns an empty `Mesh` rather than a detailed Lua error.
There is no `Mesh:is_empty` method, so test your chosen font and text during development.

### Add raised text

```lua
function execute(opts)
    local base = api.make_cube(50, 20, 2)
    local text = api.emboss_text {
        font = api.get_default_font(),
        text = opts.text,
        line_height = 8
    }

    api.project:add_object {
        mesh = base,
        other_volumes = {
            {
                mesh = text,
                type = VolumeType.Solid,
                translate = {x = 5, y = 6, z = 2}
            }
        }
    }
end
```

Text mesh coordinates depend on the font and text shape.
Use bounds from the generated text mesh to position it when those bounds are valid for your build.

### Engrave text

```lua
function execute(opts)
    local base = api.make_cube(50, 20, 3)
    local text = api.emboss_text {
        font = api.get_default_font(),
        text = opts.text,
        line_height = 8
    }

    api.project:add_object {
        mesh = base,
        other_volumes = {
            {
                mesh = text,
                type = VolumeType.Negative,
                translate = {x = 5, y = 6, z = 2.5}
            }
        }
    }
end
```

## `api.emboss_svg`

```lua
api.emboss_svg(path, depth) -> Mesh
```

Reads an SVG file relative to the command entry script and extrudes its shapes by `depth` millimeters.
The path must resolve inside the entry script's sandbox.

```text
com.example.logo/
  manifest.json
  add_logo.lua
  assets/
    logo.svg
```

```lua
function execute(opts)
    local logo = api.emboss_svg("assets/logo.svg", 1.5)

    api.project:add_object {
        mesh = logo
    }
end
```

An unsafe path raises an error.
An unreadable SVG, unsupported SVG content, or mesh-generation failure returns an empty `Mesh`.

### Combine SVG artwork with a base

```lua
function execute(opts)
    local logo = api.emboss_svg("assets/logo.svg", 1)
    local base = api.make_cube(60, 40, 2)

    api.project:add_object {
        mesh = base,
        other_volumes = {
            {
                mesh = logo,
                type = opts.engrave and VolumeType.Negative or VolumeType.Solid,
                translate = {x = 5, y = 5, z = 2}
            }
        }
    }
end
```

[Back to the API index](index.md)
