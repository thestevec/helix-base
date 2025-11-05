# Prop Protection Plugin

> **Reference**: `plugins/propprotect.lua`

The Prop Protection plugin prevents players from interacting with props they don't own. Includes spawn rate limiting and blacklist for problematic props, ensuring fair and safe prop usage.

## ⚠️ Important: Use Built-in Prop Protection

**The plugin automatically protects all spawned props**. The framework provides:
- Character-based ownership (not player-based)
- Spawn rate limiting (0.75s cooldown)
- Prop model blacklist
- Admin bypass privilege

## Protected Actions

- Physgun pickup
- Physgun reload (unfreeze)
- Properties (color, material, etc.)
- Toolgun usage
- All prevented for non-owners

## Permissions

**Bypass**: `Helix - Bypass Prop Protection`
- Admins can interact with all props
- Minimum access: admin

## Spawn Rate Limiting

**Cooldown**: 0.75 seconds between prop spawns
- Prevents prop spam
- Applied per player

## Prop Blacklist

The plugin includes blacklist of:
- Explosive props
- Large/laggy props (trains, buildings, etc.)
- Problematic models

**Non-admins cannot spawn blacklisted props**.

## Logging

Logs all spawned:
- Props (with model)
- NPCs
- SWEPs
- SENTs
- Vehicles

## Best Practices

### ✅ DO

- Respect other players' props
- Use admin bypass responsibly
- Add problematic props to blacklist

### ❌ DON'T

- Don't try to bypass protection
- Don't spam prop spawning
- Don't spawn blacklisted props

## See Also

- [Persistence Plugin](persistence.md) - Permanent props
- Source: `plugins/propprotect.lua`
