# Strength Plugin

> **Reference**: `plugins/strength/`

The Strength plugin adds a strength attribute that increases fist punch damage. Characters gain strength experience by punching other players, creating a progression system for melee combat.

## ⚠️ Important: Use Built-in Strength System

**The plugin works automatically** with the strength attribute. The framework provides:
- Automatic damage bonus based on strength attribute
- Attribute progression through use
- Configurable strength multiplier

## How It Works

### Damage Calculation

**Base fist damage + (strength attribute × multiplier)**

Example:
- Base damage: 10
- Strength: 50
- Multiplier: 0.3
- **Total damage: 10 + (50 × 0.3) = 25**

### Gaining Strength

Players gain strength attribute experience by:
- Punching other players (+0.001 per punch that hits)

## Configuration

**strengthMultiplier** (default: 0.3)
- Range: 0 to 1.0
- How much each strength point adds to damage

## Developer Hook

```lua
function PLUGIN:GetPlayerPunchDamage(client, damage, context)
    -- Modify punch damage
    context.damage = context.damage + bonusAmount
end
```

## See Also

- [Attributes System](../systems/attributes.md) - Attribute management
- [Stamina Plugin](stamina.md) - Stamina system
- Source: `plugins/strength/`
