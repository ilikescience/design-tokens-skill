# DTCG Color Module

## Contents
- [Color Token Structure](#color-token-structure)
- [Supported Color Spaces](#supported-color-spaces)
- [Color Space Cheat Sheet](#color-space-cheat-sheet)
- [Common Mistakes](#common-mistakes)
- [The `none` Keyword](#the-none-keyword)
- [Examples](#examples)

## Color Token Structure

Colors use a structured object format:

**Required properties:**
- `colorSpace` - String specifying the color model
- `components` - Array of color values (numbers or `"none"`)

**Optional properties:**
- `alpha` - Transparency value 0-1 (default: 1)
- `hex` - 6-digit CSS hex fallback in `#RRGGBB` format (excludes alpha)

## Supported Color Spaces

| Space | Value | Components | Range |
|-------|-------|------------|-------|
| sRGB | `"srgb"` | [R, G, B] | [0-1] each |
| sRGB Linear | `"srgb-linear"` | [R, G, B] | [0-1] each |
| Display P3 | `"display-p3"` | [R, G, B] | [0-1] each |
| A98 RGB | `"a98-rgb"` | [R, G, B] | [0-1] each |
| ProPhoto RGB | `"prophoto-rgb"` | [R, G, B] | [0-1] each |
| Rec 2020 | `"rec2020"` | [R, G, B] | [0-1] each |
| HSL | `"hsl"` | [H, S, L] | [0-360), [0-100], [0-100] |
| HWB | `"hwb"` | [H, W, B] | [0-360), [0-100], [0-100] |
| CIELAB | `"lab"` | [L, A, B] | [0-100], unbounded, unbounded |
| LCH | `"lch"` | [L, C, H] | [0-100], [0-∞), [0-360) |
| OKLAB | `"oklab"` | [L, A, B] | [0-1], unbounded, unbounded |
| OKLCH | `"oklch"` | [L, C, H] | [0-1], [0-∞), [0-360) |
| XYZ-D65 | `"xyz-d65"` | [X, Y, Z] | [0-1] each |
| XYZ-D50 | `"xyz-d50"` | [X, Y, Z] | [0-1] each |

**WARNING: HSL/HWB use 0-100 for saturation/lightness, NOT 0-1**

For 20% saturation in HSL, use `20`, not `0.2`. This is a common mistake because RGB-based color spaces use 0-1 ranges.

## Color Space Cheat Sheet

The same color expressed in different color spaces:

**Near-white warm gray (#fcfcfb)**
```json
{
  "warm-gray-hsl": {
    "$type": "color",
    "$value": {
      "colorSpace": "hsl",
      "components": [40, 20, 99],
      "hex": "#fcfcfb"
    },
    "$description": "HSL: hue 40°, saturation 20%, lightness 99%"
  },
  "warm-gray-srgb": {
    "$type": "color",
    "$value": {
      "colorSpace": "srgb",
      "components": [0.988, 0.988, 0.984],
      "hex": "#fcfcfb"
    },
    "$description": "sRGB: each channel 0-1"
  },
  "warm-gray-oklch": {
    "$type": "color",
    "$value": {
      "colorSpace": "oklch",
      "components": [0.988, 0.002, 90],
      "hex": "#fcfcfb"
    },
    "$description": "OKLCH: lightness 0-1, chroma 0-0.4, hue 0-360"
  }
}
```

**Brand blue (#2563eb)**
```json
{
  "brand-blue-hsl": {
    "$type": "color",
    "$value": {
      "colorSpace": "hsl",
      "components": [221, 83, 53],
      "hex": "#2563eb"
    },
    "$description": "HSL: hue 221°, saturation 83%, lightness 53%"
  },
  "brand-blue-srgb": {
    "$type": "color",
    "$value": {
      "colorSpace": "srgb",
      "components": [0.145, 0.388, 0.922],
      "hex": "#2563eb"
    },
    "$description": "sRGB: R=14.5%, G=38.8%, B=92.2% (as 0-1)"
  },
  "brand-blue-oklch": {
    "$type": "color",
    "$value": {
      "colorSpace": "oklch",
      "components": [0.546, 0.245, 262.9],
      "hex": "#2563eb"
    },
    "$description": "OKLCH: perceptually uniform, great for palette generation"
  }
}
```

### Quick Reference: Component Ranges

| Color Space | Component 1 | Component 2 | Component 3 |
|-------------|-------------|-------------|-------------|
| sRGB, Display P3 | R: 0-1 | G: 0-1 | B: 0-1 |
| **HSL, HWB** | H: 0-360 | **S/W: 0-100** | **L/B: 0-100** |
| OKLCH | L: 0-1 | C: 0-0.4 | H: 0-360 |
| OKLAB | L: 0-1 | a: ~-0.4 to 0.4 | b: ~-0.4 to 0.4 |

## Common Mistakes

### Mistake 1: Using 0-1 for HSL saturation/lightness

**Wrong:**
```json
{
  "$value": {
    "colorSpace": "hsl",
    "components": [20, 0.2, 0.99]
  }
}
```
This creates nearly black color (lightness ~1%).

**Correct:**
```json
{
  "$value": {
    "colorSpace": "hsl",
    "components": [20, 20, 99]
  }
}
```
This creates the intended near-white color (lightness 99%).

### Mistake 2: Confusing hex and component values

Hex `#ff0000` (red) in sRGB components is `[1, 0, 0]`, not `[255, 0, 0]`.

**Wrong:**
```json
{
  "$value": {
    "colorSpace": "srgb",
    "components": [255, 0, 0]
  }
}
```

**Correct:**
```json
{
  "$value": {
    "colorSpace": "srgb",
    "components": [1, 0, 0],
    "hex": "#ff0000"
  }
}
```

### Mistake 3: OKLCH chroma out of range

OKLCH chroma typically maxes around 0.4 for in-gamut colors. Values like `0.8` will be out of gamut.

**Suspicious:**
```json
{
  "$value": {
    "colorSpace": "oklch",
    "components": [0.7, 0.8, 250]
  }
}
```

**Typical range:**
```json
{
  "$value": {
    "colorSpace": "oklch",
    "components": [0.7, 0.15, 250]
  }
}
```

## The `none` Keyword

Use `"none"` for missing or non-applicable components. Different from `0`:
- In HSL, hue `0` means red
- Hue `"none"` means undefined (useful for achromatic colors during interpolation)

```json
{
  "gray-50": {
    "$type": "color",
    "$value": {
      "colorSpace": "hsl",
      "components": ["none", 0, 95]
    },
    "$description": "Achromatic gray - hue is not applicable"
  }
}
```

## Examples

### Basic sRGB with hex fallback
```json
{
  "brand-primary": {
    "$type": "color",
    "$value": {
      "colorSpace": "srgb",
      "components": [1, 0, 1],
      "alpha": 1,
      "hex": "#ff00ff"
    }
  }
}
```

### Semi-transparent shadow
```json
{
  "shadow-color": {
    "$type": "color",
    "$value": {
      "colorSpace": "srgb",
      "components": [0, 0, 0],
      "alpha": 0.5,
      "hex": "#000000"
    }
  }
}
```

### Wide gamut (Display P3)
```json
{
  "wide-gamut-blue": {
    "$type": "color",
    "$value": {
      "colorSpace": "display-p3",
      "components": [0, 0.5, 1]
    }
  }
}
```

### Perceptually uniform (OKLCH)
```json
{
  "oklch-vibrant": {
    "$type": "color",
    "$value": {
      "colorSpace": "oklch",
      "components": [0.7, 0.15, 250]
    }
  }
}
```

### Complete color palette structure
```json
{
  "color": {
    "primitive": {
      "$type": "color",
      "blue": {
        "400": {
          "$value": {"colorSpace": "srgb", "components": [0.23, 0.51, 0.96], "hex": "#3b82f6"}
        },
        "500": {
          "$value": {"colorSpace": "srgb", "components": [0.15, 0.39, 0.92], "hex": "#2563eb"}
        },
        "600": {
          "$value": {"colorSpace": "srgb", "components": [0.11, 0.31, 0.85], "hex": "#1d4ed8"}
        }
      }
    }
  }
}
```
