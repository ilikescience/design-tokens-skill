# DTCG Resolver Module

## Contents
- [Overview](#overview)
- [Document Structure](#document-structure)
- [Sets](#sets)
- [Modifiers](#modifiers)
- [Resolution Order](#resolution-order)
- [Resolver Inputs](#resolver-inputs)
- [Reference Constraints](#reference-constraints)
- [Resolution Logic](#resolution-logic)
- [Extensions](#extensions)

## Overview

The Resolver module manages design tokens across multiple contexts like themes, responsive breakpoints, and accessibility modes. It solves the "combinatorial explosion" problem by deduplicating repeated token values while enabling enumeration of all context permutations.

**Common use cases:**
- Theming (light/dark/high-contrast modes)
- Responsive sizing (mobile/tablet/desktop)
- Accessibility features (reduced motion, colorblindness accommodations)
- Brand variations

## Document Structure

A resolver is a separate JSON document with these root-level properties:

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | No | Human-readable identifier |
| `version` | string | **Yes** | Must be `"2025.10"` |
| `description` | string | No | Context and explanation |
| `sets` | Map[string, Set] | No | Named token collections |
| `modifiers` | Map[string, Modifier] | No | Conditional token collections |
| `resolutionOrder` | Array | **Yes** | Processing sequence |

**Basic structure:**
```json
{
  "name": "my-design-system",
  "version": "2025.10",
  "description": "Design tokens with theme and density support",
  "sets": {
    "core": {
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
    { "$ref": "#/sets/core" },
    { "$ref": "#/modifiers/theme" }
  ]
}
```

## Sets

Sets bundle design tokens from DTCG format files or inline declarations. The `sources` array merges tokens in order—later sources override earlier conflicts.

```json
{
  "sets": {
    "foundation": {
      "description": "Core design primitives",
      "sources": [
        { "$ref": "tokens/colors.json" },
        { "$ref": "tokens/spacing.json" },
        { "$ref": "tokens/typography.json" }
      ]
    },
    "semantic": {
      "sources": [{ "$ref": "tokens/semantic.json" }]
    },
    "components": {
      "sources": [
        { "$ref": "tokens/button.json" },
        { "$ref": "tokens/input.json" }
      ]
    }
  }
}
```

### Inline Token Declarations

```json
{
  "sets": {
    "overrides": {
      "sources": [
        {
          "color": {
            "brand": {
              "$type": "color",
              "primary": {
                "$value": { "colorSpace": "srgb", "components": [0, 0.4, 0.8] }
              }
            }
          }
        }
      ]
    }
  }
}
```

## Modifiers

Modifiers apply conditional logic through a `contexts` map. Each context key maps to an array of token sources.

**Properties:**
- `contexts` - Map of context names to source arrays (required)
- `default` - Default context key when no input provided (optional)

```json
{
  "modifiers": {
    "theme": {
      "contexts": {
        "light": [{ "$ref": "themes/light.json" }],
        "dark": [{ "$ref": "themes/dark.json" }],
        "high-contrast": [
          { "$ref": "themes/light.json" },
          { "$ref": "themes/high-contrast.json" }
        ]
      },
      "default": "light"
    },
    "density": {
      "contexts": {
        "comfortable": [{ "$ref": "density/comfortable.json" }],
        "compact": [{ "$ref": "density/compact.json" }]
      },
      "default": "comfortable"
    },
    "motion": {
      "contexts": {
        "full": [{ "$ref": "motion/full.json" }],
        "reduced": [{ "$ref": "motion/reduced.json" }]
      },
      "default": "full"
    }
  }
}
```

### Composing Contexts

Multiple sources merge in order, enabling theme composition:
```json
{
  "dark-high-contrast": [
    { "$ref": "themes/dark.json" },
    { "$ref": "themes/high-contrast-overlay.json" }
  ]
}
```

## Resolution Order

The `resolutionOrder` array defines processing sequence. Later entries override earlier conflicts.

### Reference-based
```json
{
  "resolutionOrder": [
    { "$ref": "#/sets/foundation" },
    { "$ref": "#/sets/semantic" },
    { "$ref": "#/modifiers/theme" },
    { "$ref": "#/modifiers/density" },
    { "$ref": "#/sets/components" }
  ]
}
```

### Inline declarations
```json
{
  "resolutionOrder": [
    { "$ref": "#/sets/core" },
    {
      "type": "modifier",
      "name": "brand",
      "contexts": {
        "default": [{ "$ref": "brands/default.json" }],
        "partner": [{ "$ref": "brands/partner.json" }]
      },
      "default": "default"
    }
  ]
}
```

## Resolver Inputs

Tools accept inputs as a JSON object mapping modifier names to context values:

```json
{
  "theme": "dark",
  "density": "compact",
  "motion": "reduced"
}
```

**Validation rules:**
- Every modifier without a `default` must have input
- Input values must match defined context keys
- All input values must be strings
- Tools should be case-insensitive

**Example resolution:**
```javascript
const resolver = loadResolver("tokens.resolver.json");
const inputs = { "theme": "dark", "density": "compact" };
const tokens = resolve(resolver, inputs);
```

## Reference Constraints

1. **Only `resolutionOrder` may reference modifiers** - Sets and modifiers cannot reference other modifiers
2. **Nothing may reference `resolutionOrder` items** - The array is not referenceable
3. **Circular references prohibited** - No self-references or cyclic chains
4. **Same-document references required** - Support `#/sets/name` and `#/modifiers/name`

```json
// VALID
{
  "resolutionOrder": [{ "$ref": "#/modifiers/theme" }]
}

// INVALID - set references modifier
{
  "sets": {
    "broken": {
      "sources": [{ "$ref": "#/modifiers/theme" }]
    }
  }
}
```

## Resolution Logic

Four stages:
1. **Input Validation** - Verify inputs match modifier definitions
2. **Ordering** - Flatten sets and modifiers per `resolutionOrder` and selected contexts
3. **Alias Resolution** - Resolve token references after ordering
4. **Final Resolution** - Produce final token structure

**Conflict resolution:** "Last occurrence wins" - applies within set source arrays and across resolution order.

## Extensions

Attach vendor-specific metadata using `$extensions`:

```json
{
  "sets": {
    "core": {
      "sources": [{ "$ref": "tokens/core.json" }],
      "$extensions": {
        "com.figma": {
          "libraryId": "abc123"
        }
      }
    }
  }
}
```
