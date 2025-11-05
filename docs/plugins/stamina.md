# Stamina Plugin

> **Reference**: `plugins/stamina/`

The Stamina plugin adds a stamina system that drains when running and regenerates when walking or crouching. Players cannot sprint when out of breath, encouraging tactical movement and rest periods.

## ⚠️ Important: Use Built-in Stamina

**The stamina system works automatically**. The framework provides:
- Automatic drain when sprinting
- Automatic regeneration when walking/crouching
- Breathing state when stamina depleted
- HUD bar display
- Attribute integration (endurance affects stamina)

## Configuration

- `staminaDrain` (default: 1) - Drain per tick when sprinting
- `staminaRegeneration` (default: 1.75) - Regen per tick when walking
- `staminaCrouchRegeneration` (default: 2) - Regen per tick when crouching
- `punchStamina` (default: 10) - Stamina cost for punching

## Behavior

- **Sprinting**: Drains stamina
- **Walking**: Regenerates stamina normally
- **Crouching**: Regenerates stamina faster
- **Out of Breath**: Cannot sprint until 50% stamina recovered

## Developer Functions

```lua
-- Server-side only
client:RestoreStamina(amount)  -- Add stamina
client:ConsumeStamina(amount)  -- Remove stamina
```

## Hooks

```lua
function PLUGIN:AdjustStaminaOffset(client, offset)
    -- Modify stamina change rate
    return offset * 2  -- Example: Double regen/drain
end

function PLUGIN:PlayerStaminaLost(client)
    -- Called when stamina reaches 0
end

function PLUGIN:PlayerStaminaGained(client)
    -- Called when stamina recovers to 50%
end
```

## See Also

- [Attributes System](../systems/attributes.md) - Endurance attribute
- Source: `plugins/stamina/`
