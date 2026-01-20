# DTCG Format Specification

## Contents
- [Fundamental Structure](#fundamental-structure)
- [Required and Optional Properties](#required-and-optional-properties)
- [Token Types](#token-types)
- [Groups and Organization](#groups-and-organization)
- [Type Inheritance](#type-inheritance)
- [References and Aliasing](#references-and-aliasing)
- [Group Extension](#group-extension)
- [File Format Conventions](#file-format-conventions)
- [Validation Requirements](#validation-requirements)

## Fundamental Structure

Design tokens are JSON files with a name-value structure where:
- An object with a `$value` property is a token
- Objects without `$value` are groups
- All special properties are prefixed with `$`

## Required and Optional Properties

**Required:**
- `$value` - The token's actual value

**Optional:**
- `$type` - Categorizes the token (color, dimension, etc.)
- `$description` - Plain text explanation
- `$deprecated` - Boolean or string marking obsolescence
- `$extensions` - Vendor-specific metadata using reverse domain notation (e.g., `com.example.metadata`)

## Token Types

### Atomic Types

**1. Color** - See [color.md](color.md) for full details.

**2. Dimension** - Object with value and unit:
```json
{"value": 16, "unit": "px"}
```
Units limited to `"px"` or `"rem"`.

**3. Font Family** - String or array:
```json
"$value": "Helvetica"
// or
"$value": ["Helvetica", "Arial", "sans-serif"]
```

**4. Font Weight** - Number (1-1000) or predefined strings:
- `"thin"` (100), `"extralight"` (200), `"light"` (300)
- `"normal"` (400), `"medium"` (500), `"semibold"` (600)
- `"bold"` (700), `"extrabold"` (800), `"black"` (900)

**5. Duration** - Object with value and unit:
```json
{"value": 200, "unit": "ms"}
```
Units limited to `"ms"` or `"s"`.

**6. Cubic Bézier** - Array of four numbers `[P1x, P1y, P2x, P2y]`:
```json
[0.42, 0, 0.58, 1]
```
**Constraint:** P1x and P2x (positions 0 and 2) must be in range [0-1].

**7. Number** - JSON numeric value for unitless quantities.

### Composite Types

**Stroke Style** - String or object:
```json
"$value": "dashed"
// or
"$value": {
  "dashArray": [
    {"value": 0.5, "unit": "rem"},
    {"value": 0.25, "unit": "rem"}
  ],
  "lineCap": "round"
}
```

**Border** - Combined properties:
```json
{
  "color": "{colors.primary}",
  "width": {"value": 2, "unit": "px"},
  "style": "solid"
}
```

**Shadow** - Single or array of shadow objects:
```json
{
  "color": {
    "colorSpace": "srgb",
    "components": [0, 0, 0],
    "alpha": 0.5
  },
  "offsetX": {"value": 0, "unit": "px"},
  "offsetY": {"value": 4, "unit": "px"},
  "blur": {"value": 8, "unit": "px"},
  "spread": {"value": 0, "unit": "px"},
  "inset": false
}
```

**Gradient** - Array of color stops:
```json
[
  {
    "color": {"colorSpace": "srgb", "components": [1, 1, 1]},
    "position": 0
  },
  {
    "color": {"colorSpace": "srgb", "components": [0, 0, 0]},
    "position": 1
  }
]
```

**Typography** - Combined text properties:
```json
{
  "fontFamily": "{fonts.body}",
  "fontSize": {"value": 16, "unit": "px"},
  "fontWeight": 400,
  "fontStyle": "normal",
  "letterSpacing": {"value": 0, "unit": "px"},
  "lineHeight": 1.5
}
```
All properties optional: `fontFamily`, `fontSize`, `fontWeight`, `fontStyle`, `letterSpacing`, `lineHeight`.

**Transition** - Animation properties:
```json
{
  "duration": {"value": 200, "unit": "ms"},
  "delay": {"value": 0, "unit": "ms"},
  "timingFunction": [0.4, 0, 0.2, 1]
}
```

## Groups and Organization

Groups organize tokens hierarchically:

```json
{
  "colors": {
    "$type": "color",
    "$description": "Brand colors",
    "primary": {
      "$value": {"colorSpace": "srgb", "components": [0.2, 0.4, 0.8]}
    },
    "secondary": {
      "$value": {"colorSpace": "srgb", "components": [0.8, 0.2, 0.4]}
    }
  }
}
```

**Key Points:**
- Groups are unordered - tools MUST NOT infer meaning from order
- Groups can have `$type` which children inherit
- Groups support all optional properties

### The `$root` Token

Use `$root` when a group needs both a base token and variants:

```json
{
  "color": {
    "accent": {
      "$type": "color",
      "$root": {
        "$value": {"colorSpace": "srgb", "components": [0.2, 0.5, 1]},
        "$description": "Default accent color"
      },
      "light": {
        "$value": {"colorSpace": "srgb", "components": [0.4, 0.7, 1]}
      },
      "dark": {
        "$value": {"colorSpace": "srgb", "components": [0.1, 0.3, 0.8]}
      }
    }
  }
}
```

This allows `color.accent` to resolve to `$root` while `color.accent.light` and `color.accent.dark` provide variants.

## Type Inheritance

Type resolution precedence:
1. Token's explicit `$type`
2. Closest parent group's `$type`
3. Invalid if unresolvable

## References and Aliasing

### Curly Brace Syntax (Primary method)

References using dot-separated paths:

```json
{
  "colors": {
    "primary": {
      "$type": "color",
      "$value": {"colorSpace": "srgb", "components": [0, 0.4, 0.8]}
    }
  },
  "button": {
    "background": {
      "$type": "color",
      "$value": "{colors.primary}"
    }
  }
}
```

### JSON Pointer Syntax (`$ref`)

RFC 6901 JSON Pointer for property-level access:

```json
{
  "blue": {
    "$type": "color",
    "$value": {"colorSpace": "okhsl", "components": [0.733, 0.8, 0.5]}
  },
  "blue-hue": {
    "$type": "number",
    "$ref": "#/blue/$value/components/0"
  }
}
```

**Formats:**
- Same-document: `{ "$ref": "#/path/to/token" }`
- External files: `{ "$ref": "path/to/file.json" }`
- External with fragment: `{ "$ref": "tokens.json#/colors/primary" }`

**Special characters:** Escape `/` as `~1` per RFC 6901.

### Extending References

References can include overriding properties:

```json
{
  "base-shadow": {
    "$type": "shadow",
    "$value": {
      "color": {"colorSpace": "srgb", "components": [0, 0, 0], "alpha": 0.3},
      "offsetX": {"value": 0, "unit": "px"},
      "offsetY": {"value": 2, "unit": "px"},
      "blur": {"value": 4, "unit": "px"}
    }
  },
  "elevated-shadow": {
    "$ref": "#/base-shadow",
    "offsetY": {"value": 8, "unit": "px"},
    "blur": {"value": 16, "unit": "px"}
  }
}
```

### Critical Constraints

1. **Circular references prohibited** - Direct, mutual, and parent references are invalid
2. **Naming restrictions** - Names cannot begin with `$` or contain `{`, `}`, `.`
3. **Resolution requirements** - All references must resolve; tools MUST reject circular references

## Group Extension

```json
{
  "button": {
    "$type": "color",
    "background": {
      "$value": {"colorSpace": "srgb", "components": [0, 0.4, 0.8]}
    },
    "text": {
      "$value": {"colorSpace": "srgb", "components": [1, 1, 1]}
    }
  },
  "button-primary": {
    "$extends": "{button}",
    "background": {
      "$value": {"colorSpace": "srgb", "components": [1, 0, 0.4]}
    }
  }
}
```

Local definitions override inherited ones at the same path.

## File Format Conventions

**Token files:**
- MIME type: `application/design-tokens+json` (preferred) or `application/json`
- Extensions: `.tokens` or `.tokens.json` (recommended)
- Encoding: UTF-8

**Resolver files:**
- MIME type: `application/json`
- Extensions: `.resolver.json` (recommended)
- Encoding: UTF-8

## Validation Requirements

Tools must validate:
- All tokens have resolvable types
- Values match declared type syntax
- No circular references
- Required properties for composite types present
- Dimension units are `"px"` or `"rem"`
- Duration units are `"ms"` or `"s"`
- Font weights are 1-1000 or predefined strings
- Cubic Bézier P1x and P2x values are in range [0-1]
- Color components match their color space's expected ranges
- Token/group names do not begin with `$` or contain `{`, `}`, `.`
