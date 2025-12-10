# Common Use Cases and Examples

## Contents
- [Multi-Platform Design System](#use-case-1-multi-platform-design-system)
- [Dynamic Theming with Brand Variations](#use-case-2-dynamic-theming-with-brand-variations)
- [Responsive Design Tokens](#use-case-3-responsive-design-tokens)
- [Animation System](#use-case-4-animation-system)
- [Elevation and Shadow System](#use-case-5-elevation-and-shadow-system)

---

## Use Case 1: Multi-Platform Design System

**Problem:** Need tokens that work across web, iOS, and Android.

**Solution:**
```json
{
  "spacing": {
    "$type": "dimension",
    "$description": "Base 8pt grid system",
    "scale": {
      "xs": {"$value": {"value": 4, "unit": "px"}},
      "sm": {"$value": {"value": 8, "unit": "px"}},
      "md": {"$value": {"value": 16, "unit": "px"}},
      "lg": {"$value": {"value": 24, "unit": "px"}},
      "xl": {"$value": {"value": 32, "unit": "px"}}
    }
  },
  "typography": {
    "body": {
      "$type": "typography",
      "$value": {
        "fontFamily": ["system-ui", "-apple-system", "BlinkMacSystemFont"],
        "fontSize": {"value": 16, "unit": "px"},
        "fontWeight": 400,
        "lineHeight": 1.5
      },
      "$extensions": {
        "com.apple.swiftui": {
          "font": ".body"
        },
        "com.android": {
          "textAppearance": "TextAppearance.Material3.BodyMedium"
        }
      }
    }
  }
}
```

**Key points:**
- Use `$extensions` for platform-specific mappings
- Stick to `px` units in source; transform for each platform at build time
- System font stacks provide native feel across platforms

---

## Use Case 2: Dynamic Theming with Brand Variations

**Problem:** Multiple brand themes with shared structure.

**Solution using Resolvers:**

### File structure
```
tokens/
├── resolver.json
├── brand-base.tokens.json
└── brands/
    ├── partner-a.tokens.json
    └── partner-b.tokens.json
```

### Base tokens (brand-base.tokens.json)
```json
{
  "brand": {
    "$type": "color",
    "primary": {
      "$value": {"colorSpace": "srgb", "components": [0, 0.4, 0.8], "hex": "#0066cc"}
    },
    "secondary": {
      "$value": {"colorSpace": "srgb", "components": [1, 0.4, 0], "hex": "#ff6600"}
    }
  },
  "spacing": {
    "$type": "dimension",
    "unit": {"$value": {"value": 8, "unit": "px"}}
  }
}
```

### Partner A overrides (brands/partner-a.tokens.json)
```json
{
  "brand": {
    "$type": "color",
    "primary": {
      "$value": {"colorSpace": "srgb", "components": [0.8, 0, 0.4], "hex": "#cc0066"}
    },
    "secondary": {
      "$value": {"colorSpace": "srgb", "components": [0, 0.8, 0.4], "hex": "#00cc66"}
    }
  }
}
```

### Resolver configuration
```json
{
  "name": "multi-brand-system",
  "version": "2025.10",
  "sets": {
    "core": {
      "sources": [{ "$ref": "tokens/brand-base.tokens.json" }]
    }
  },
  "modifiers": {
    "brand": {
      "contexts": {
        "default": [],
        "partner-a": [{ "$ref": "tokens/brands/partner-a.tokens.json" }],
        "partner-b": [{ "$ref": "tokens/brands/partner-b.tokens.json" }]
      },
      "default": "default"
    }
  },
  "resolutionOrder": [
    { "$ref": "#/sets/core" },
    { "$ref": "#/modifiers/brand" }
  ]
}
```

### Build commands
```bash
terrazzo build --resolver resolver.json --output default.css
terrazzo build --resolver resolver.json --input brand=partner-a --output partner-a.css
terrazzo build --resolver resolver.json --input brand=partner-b --output partner-b.css
```

**Key points:**
- Token paths stay consistent (`brand.primary`) across all brands
- Only override what changes; shared tokens remain in core
- Easy to add new brands without touching existing files

---

## Use Case 3: Responsive Design Tokens

**Problem:** Different values for different screen sizes.

**Solution:**
```json
{
  "breakpoint": {
    "mobile": {
      "typography": {
        "heading": {
          "$type": "typography",
          "$value": {
            "fontSize": {"value": 24, "unit": "px"},
            "lineHeight": 1.2
          }
        }
      },
      "spacing": {
        "container": {
          "$type": "dimension",
          "$value": {"value": 16, "unit": "px"}
        }
      }
    },
    "desktop": {
      "typography": {
        "heading": {
          "$type": "typography",
          "$value": {
            "fontSize": {"value": 36, "unit": "px"},
            "lineHeight": 1.3
          }
        }
      },
      "spacing": {
        "container": {
          "$type": "dimension",
          "$value": {"value": 24, "unit": "px"}
        }
      }
    }
  },
  "$extensions": {
    "com.example.responsive": {
      "breakpoints": {
        "mobile": "max-width: 768px",
        "desktop": "min-width: 769px"
      }
    }
  }
}
```

**Alternative: Using resolvers for responsive tokens**
```json
{
  "modifiers": {
    "viewport": {
      "contexts": {
        "mobile": [{ "$ref": "tokens/responsive/mobile.json" }],
        "tablet": [{ "$ref": "tokens/responsive/tablet.json" }],
        "desktop": [{ "$ref": "tokens/responsive/desktop.json" }]
      },
      "default": "desktop"
    }
  }
}
```

---

## Use Case 4: Animation System

**Problem:** Consistent motion design across application.

**Solution:**
```json
{
  "motion": {
    "duration": {
      "$type": "duration",
      "instant": {"$value": {"value": 0, "unit": "ms"}},
      "fast": {"$value": {"value": 100, "unit": "ms"}},
      "normal": {"$value": {"value": 200, "unit": "ms"}},
      "slow": {"$value": {"value": 300, "unit": "ms"}},
      "slower": {"$value": {"value": 500, "unit": "ms"}}
    },
    "easing": {
      "$type": "cubicBezier",
      "linear": {"$value": [0, 0, 1, 1]},
      "ease": {"$value": [0.25, 0.1, 0.25, 1]},
      "ease-in": {"$value": [0.42, 0, 1, 1]},
      "ease-out": {"$value": [0, 0, 0.58, 1]},
      "ease-in-out": {"$value": [0.42, 0, 0.58, 1]}
    },
    "transition": {
      "$type": "transition",
      "default": {
        "$value": {
          "duration": "{motion.duration.normal}",
          "delay": {"value": 0, "unit": "ms"},
          "timingFunction": "{motion.easing.ease}"
        }
      },
      "fade": {
        "$value": {
          "duration": "{motion.duration.fast}",
          "delay": {"value": 0, "unit": "ms"},
          "timingFunction": "{motion.easing.ease-in-out}"
        }
      }
    }
  }
}
```

### Reduced motion support with resolvers
```json
{
  "modifiers": {
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

**motion/reduced.json:**
```json
{
  "motion": {
    "duration": {
      "$type": "duration",
      "instant": {"$value": {"value": 0, "unit": "ms"}},
      "fast": {"$value": {"value": 0, "unit": "ms"}},
      "normal": {"$value": {"value": 0, "unit": "ms"}},
      "slow": {"$value": {"value": 0, "unit": "ms"}},
      "slower": {"$value": {"value": 0, "unit": "ms"}}
    }
  }
}
```

---

## Use Case 5: Elevation and Shadow System

**Problem:** Consistent depth hierarchy.

**Solution:**
```json
{
  "elevation": {
    "$type": "shadow",
    "level-1": {
      "$description": "Cards, buttons",
      "$value": {
        "color": {"colorSpace": "srgb", "components": [0, 0, 0], "alpha": 0.1},
        "offsetX": {"value": 0, "unit": "px"},
        "offsetY": {"value": 1, "unit": "px"},
        "blur": {"value": 3, "unit": "px"},
        "spread": {"value": 0, "unit": "px"}
      }
    },
    "level-2": {
      "$description": "Dropdowns, popovers",
      "$value": {
        "color": {"colorSpace": "srgb", "components": [0, 0, 0], "alpha": 0.2},
        "offsetX": {"value": 0, "unit": "px"},
        "offsetY": {"value": 4, "unit": "px"},
        "blur": {"value": 8, "unit": "px"},
        "spread": {"value": 0, "unit": "px"}
      }
    },
    "level-3": {
      "$description": "Modals, dialogs",
      "$value": [
        {
          "color": {"colorSpace": "srgb", "components": [0, 0, 0], "alpha": 0.1},
          "offsetX": {"value": 0, "unit": "px"},
          "offsetY": {"value": 8, "unit": "px"},
          "blur": {"value": 16, "unit": "px"},
          "spread": {"value": 0, "unit": "px"}
        },
        {
          "color": {"colorSpace": "srgb", "components": [0, 0, 0], "alpha": 0.05},
          "offsetX": {"value": 0, "unit": "px"},
          "offsetY": {"value": 16, "unit": "px"},
          "blur": {"value": 32, "unit": "px"},
          "spread": {"value": 0, "unit": "px"}
        }
      ]
    }
  }
}
```

**Key points:**
- Level 3 uses an array for layered shadows (more realistic depth)
- Shadow alpha increases with elevation
- Blur and offset scale proportionally
- Descriptions document intended use cases
