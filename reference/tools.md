# Tool Integration

## Contents
- [jq for Token Manipulation](#jq-for-token-manipulation)
- [JSONata for Complex Transformations](#jsonata-for-complex-transformations)
- [Figma Variables Export](#figma-variables-export)
- [Terrazzo for Advanced Operations](#terrazzo-for-advanced-operations)

## jq for Token Manipulation

jq is excellent for querying and transforming JSON token files.

### Extract all color tokens
```bash
jq '.. | objects | select(has("$value") and (."$type" // "" | test("color")))' tokens.json
```

### Get all token names at any depth
```bash
jq -r '.. | objects | select(has("$value")) | keys[]' tokens.json
```

### Transform dimension units from px to rem (16px base)
```bash
jq 'walk(if type == "object" and has("$value") and ."$type" == "dimension" then
  .["$value"].unit = "rem" | .["$value"].value = (.["$value"].value / 16)
  else . end)' tokens.json
```

### Flatten nested token structure
```bash
jq -r 'paths(has("$value")) as $p | "\($p | join(".")): \(getpath($p).$value)"' tokens.json
```

### List all tokens with their types
```bash
jq -r '
  [paths(has("$value"))] as $paths |
  $paths[] | . as $p |
  "\($p | join(".")): \(getpath($p) | ."$type" // "untyped")"
' tokens.json
```

### Extract tokens by specific type
```bash
jq '[.. | objects | select(."$type" == "dimension")]' tokens.json
```

### Find all token references (aliases)
```bash
jq -r '.. | objects | select(has("$value")) | ."$value" | strings | select(startswith("{"))' tokens.json
```

## JSONata for Complex Transformations

JSONata provides powerful transformation capabilities.

### Extract tokens by type
```jsonata
$..{
  $ ~> $type() = "object" and $keys() = "$value" ? {
    "name": $keys($)[0],
    "value": $.$value,
    "type": $.$type
  }
}
```

### Resolve all aliases to absolute values
```jsonata
$resolve := function($ref, $root) {
  $ref ~> /\{([^}]+)\}/ ? (
    $path := $split($1, '.');
    $lookup($root, $path).$value
  ) : $ref
}
```

### Convert to CSS custom properties
```jsonata
$..{
  $ ~> $type() = "object" and $keys() = "$value" ? (
    "--" & $join($keys($parent), "-") & ": " & $.$value
  )
}
```

### Generate a flat token map
```jsonata
$flatten := function($obj, $prefix) {
  $spread($obj) ~> $map(function($v, $k) {
    $v.$value ? { ($prefix ? $prefix & "." : "") & $k: $v.$value } :
    $type($v) = "object" ? $flatten($v, ($prefix ? $prefix & "." : "") & $k)
  }) ~> $merge()
};
$flatten($)
```

## Figma Variables Export

Figma supports exporting Variables in DTCG format. Here's how to work with these exports.

### What Figma Exports

Figma's native DTCG export produces:
- Colors as structured objects with `colorSpace: "srgb"` and components in 0-1 range
- Dimensions with `value` and `unit`
- References using `{path.to.token}` syntax
- Mode support through separate files or mode-prefixed tokens

### Export Steps

1. In Figma, open a file with Variables defined
2. Go to **Plugins > Design Tokens** or use Figma's built-in export
3. Select DTCG format
4. Export as JSON

### Common Post-Export Cleanup

Figma exports often need adjustments:

**1. Add hex fallbacks for compatibility:**
```bash
# Use jq to add hex values to color tokens
jq 'walk(if type == "object" and .colorSpace == "srgb" and .components then
  . + {"hex": "#" + (
    [.components[] * 255 | floor |
     if . < 16 then "0" else "" end +
     (. | floor | tostring | .[0:2])] | join("")
  )}
  else . end)' figma-export.json > tokens.json
```

**2. Restructure flat exports into layers:**

Figma may export flat structures. Reorganize into primitives/semantic layers:

```bash
# Extract primitive colors (those without references)
jq '{color: {primitive: .color | with_entries(select(.value."$value" | type == "object"))}}' \
  figma-export.json > primitives.tokens.json

# Extract semantic colors (those with references)
jq '{color: {semantic: .color | with_entries(select(.value."$value" | type == "string"))}}' \
  figma-export.json > semantic.tokens.json
```

**3. Fix naming conventions:**

Convert Figma's naming to kebab-case:
```bash
jq 'walk(if type == "object" then with_entries(.key |= gsub(" "; "-") | .key |= ascii_downcase) else . end)' \
  figma-export.json > tokens.json
```

### Validating Figma Exports

Check for common issues:

```bash
# Check all color tokens have valid colorSpace
jq '.. | objects | select(."$type" == "color") | select(.["$value"].colorSpace == null)' tokens.json

# Find unresolved references
jq -r '.. | strings | select(startswith("{")) | select(endswith("}"))' tokens.json | sort -u

# Verify no invalid characters in token names
jq -r 'paths(has("$value")) | join(".")' tokens.json | grep -E '[\{\}\.]'
```

### Figma Mode Handling

If Figma exports modes (light/dark), you'll get either:

**Option A: Separate files per mode**
```
figma-export/
├── light.tokens.json
└── dark.tokens.json
```

Add `$extensions.mode` to each file for Terrazzo compatibility.

**Option B: Mode-prefixed tokens**
```json
{
  "light/background": { "$value": {...} },
  "dark/background": { "$value": {...} }
}
```

Split these into separate files:
```bash
jq 'with_entries(select(.key | startswith("light/"))) |
    with_entries(.key |= ltrimstr("light/"))' export.json > light.tokens.json
```

## Terrazzo for Advanced Operations

Terrazzo (https://terrazzo.app/) is a comprehensive token transformation tool.

### Installation
```bash
npm install -D @terrazzo/cli @terrazzo/plugin-css
```

### DTCG Resolver Support

Terrazzo supports the DTCG Resolver specification for managing tokens across multiple contexts (themes, accessibility modes, responsive breakpoints, etc.). Instead of listing token files individually, you can use a resolver configuration.

### Basic Configuration with Resolver (terrazzo.config.mjs)

```javascript
import { defineConfig } from "@terrazzo/cli";
import pluginCSS from "@terrazzo/plugin-css";

export default defineConfig({
  tokens: "./tokens.resolver.json",  // Point to resolver file
  outDir: "./dist/",
  plugins: [
    pluginCSS({
      filename: "tokens.css",
      colorFormat: "hex",
    }),
  ],
});
```

### Resolver File Structure (tokens.resolver.json)

```json
{
  "version": "2025.10",
  "sets": {
    "foundation": {
      "sources": [
        { "$ref": "tokens/primitives.json" },
        { "$ref": "tokens/semantic.json" }
      ]
    }
  },
  "modifiers": {
    "theme": {
      "contexts": {
        "light": [{ "$ref": "themes/light.json" }],
        "dark": [{ "$ref": "themes/dark.json" }]
      },
      "default": "light"
    }
  },
  "resolutionOrder": [
    { "$ref": "#/sets/foundation" },
    { "$ref": "#/modifiers/theme" }
  ]
}
```

### Context Selectors for CSS Output

Map resolver contexts to CSS selectors using `contextSelectors` in the CSS plugin:

```javascript
import { defineConfig } from "@terrazzo/cli";
import pluginCSS from "@terrazzo/plugin-css";

export default defineConfig({
  tokens: "./tokens.resolver.json",
  outDir: "./dist/",
  plugins: [
    pluginCSS({
      filename: "tokens.css",
      contextSelectors: [
        { selector: ":root", context: { theme: "light" } },
        { selector: "[data-theme='dark']", context: { theme: "dark" } },
        { selector: "@media (prefers-color-scheme: dark)", context: { theme: "dark" } },
      ],
    }),
  ],
});
```

### Multiple Context Dimensions

Resolvers support multiple independent modifier dimensions:

```json
{
  "modifiers": {
    "theme": {
      "contexts": {
        "light": [{ "$ref": "themes/light.json" }],
        "dark": [{ "$ref": "themes/dark.json" }]
      },
      "default": "light"
    },
    "density": {
      "contexts": {
        "comfortable": [{ "$ref": "density/comfortable.json" }],
        "compact": [{ "$ref": "density/compact.json" }]
      },
      "default": "comfortable"
    }
  }
}
```

Map combinations in CSS:

```javascript
contextSelectors: [
  { selector: ":root", context: { theme: "light", density: "comfortable" } },
  { selector: "[data-theme='dark']", context: { theme: "dark" } },
  { selector: ".compact", context: { density: "compact" } },
]
```

### Legacy Mode Support

Terrazzo maintains backwards compatibility with the older `$extensions.mode` syntax through a special `tzMode` namespace mapping. If you have existing token files using this pattern, they will continue to work:

**tokens/themes/light.tokens.json** (legacy approach)
```json
{
  "$extensions": {
    "mode": "light"
  },
  "color": {
    "background": {
      "$type": "color",
      "page": {
        "$value": { "colorSpace": "srgb", "components": [1, 1, 1], "hex": "#ffffff" }
      }
    }
  }
}
```

For new projects, the resolver approach is recommended as it provides better organization and follows the DTCG specification.

### colorFormat Options

Control how colors are output in CSS:

| Value | Output | Use Case |
|-------|--------|----------|
| `"hex"` | `#2563eb` | Maximum compatibility |
| `"rgb"` | `rgb(37 99 235)` | Standard, good compatibility |
| `"hsl"` | `hsl(221 83% 53%)` | Easy manual tweaking |
| `"oklch"` | `oklch(0.55 0.25 263)` | Perceptually uniform, modern browsers |
| `"p3"` | `color(display-p3 0.15 0.39 0.92)` | Wide gamut displays |

### CLI Commands

```bash
# Build tokens
npx terrazzo build

# Build with watch mode
npx terrazzo build --watch

# Check for errors (no output)
npx terrazzo check
```

### Generated CSS Output

With the modeSelectors config above, Terrazzo generates:

```css
:root,
[data-theme='light'] {
  --color-background-page: #ffffff;
}

[data-theme='dark'],
@media (prefers-color-scheme: dark) {
  --color-background-page: #121212;
}
```

### Troubleshooting

**"Token not found" errors:**
- Ensure all token files are listed in `tokens` array
- Check that references like `{color.primary}` resolve to existing tokens
- Verify file paths are relative to config file location

**Colors look wrong:**
- Check component ranges (HSL uses 0-100, sRGB uses 0-1)
- Verify `colorFormat` matches your needs
- Check if wide-gamut colors are being clamped to sRGB

**Themes not switching:**
- Verify `$extensions.mode` is set in theme files
- Check `modeSelectors` mapping matches your mode names
- Ensure CSS selectors match your HTML structure
