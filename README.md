# Design Tokens Skill for Claude Code

A Claude Code skill that provides expert guidance for working with design tokens following the [Design Tokens Community Group (DTCG)](https://www.designtokens.org/) specification.

## What This Skill Does

When activated, Claude becomes an expert in:

- **DTCG Format** - Token structure, types, validation, and best practices
- **Color Spaces** - sRGB, Display P3, OKLCH, and proper color value formatting
- **References & Aliasing** - Token references and dependency resolution
- **Resolvers** - Theming with modifiers, contexts, and multi-platform support
- **Tools** - jq queries, JSONata transforms, Terrazzo configuration, Figma exports

## Installation

### Skills Marketplace (recommended)

```bash
npx skills add https://github.com/ilikescience/design-tokens-skill --skill design-tokens
```

### Manual

Copy the entire skill folder to your Claude Code skills directory:

```bash
# User-level skill (available in all projects)
cp -r . ~/.claude/skills/design-tokens/

# Or project-level skill
cp -r . .claude/skills/design-tokens/
```

The skill requires the full folder structure since SKILL.md references documentation in `reference/`, `examples/`, and `fixtures/`.

## Usage

The skill activates automatically when you ask Claude about design tokens. Examples:

```
"Help me convert these CSS variables to DTCG format"
"Set up a resolver for light and dark themes"
"Validate my token file structure"
"Configure Terrazzo for multi-platform output"
```

## Repository Structure

```
├── SKILL.md              # Main skill definition
├── reference/            # Detailed reference docs
│   ├── format.md         # Token types and structure
│   ├── color.md          # Color spaces and components
│   ├── resolver.md       # Sets, modifiers, resolution
│   └── tools.md          # jq, JSONata, Terrazzo, Figma
├── examples/             # Common patterns and use cases
├── fixtures/             # Example token files
│   ├── primitives.tokens.json
│   ├── semantic.tokens.json
│   ├── resolver.json
│   └── themes/
└── evaluations/          # Test cases for skill validation
```

## Resources

- [DTCG Format Spec](https://www.designtokens.org/tr/drafts/format/)
- [DTCG Color Module](https://www.designtokens.org/tr/drafts/color/)
- [DTCG Resolver Module](https://www.designtokens.org/tr/drafts/resolver/)
- [Terrazzo CLI](https://terrazzo.app/)
