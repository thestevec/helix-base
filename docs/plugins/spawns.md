# Spawns Plugin

> **Reference**: `plugins/spawns.lua`

The Spawns plugin allows administrators to define custom spawn points for factions and classes. Players spawn at faction/class-specific locations instead of default map spawns, enabling proper spawn separation and roleplay area assignment.

## ⚠️ Important: Use Built-in Spawn System

**Always use SpawnAdd/SpawnRemove commands** to manage spawns. The framework provides:
- Faction-specific spawn points
- Class-specific spawn points within factions
- Random selection from multiple spawn points
- Persistent storage across restarts
- Automatic spawning on PlayerLoadout

## Using Spawns

### Adding Spawn Points

**Reference**: `plugins/spawns.lua:53-117`

```lua
-- Command usage:
/SpawnAdd <faction> [class]

-- Examples:
/SpawnAdd citizen              -- Default spawn for all citizen classes
/SpawnAdd citizen medic        -- Specific spawn for medic class
/SpawnAdd "Civil Protection"   -- Faction with spaces
```

**How it works**:
1. Stand where you want the spawn point
2. Use /SpawnAdd with faction name
3. Optionally specify class name
4. Spawn point saved at your position

### Removing Spawn Points

**Reference**: `plugins/spawns.lua:119-147`

```lua
-- Command usage:
/SpawnRemove [radius]

-- Examples:
/SpawnRemove          -- Remove spawns within 120 units
/SpawnRemove 200      -- Remove spawns within 200 units
```

## Complete Example

```lua
-- Setup spawns programmatically
hook.Add("InitPostEntity", "SetupSpawns", function()
    timer.Simple(2, function()
        local plugin = ix.plugin.list["spawns"]

        -- Citizen default spawn
        plugin.spawns["citizen"] = plugin.spawns["citizen"] or {}
        plugin.spawns["citizen"]["default"] = {
            Vector(100, 200, 64),
            Vector(150, 200, 64),
            Vector(100, 250, 64)
        }

        -- Medic specific spawn
        plugin.spawns["citizen"]["medic"] = {
            Vector(500, 600, 64)
        }

        plugin:SaveSpawns()
    end)
end)
```

## Best Practices

### ✅ DO

- Set default spawns for each faction
- Add multiple spawn points to prevent clustering
- Use class-specific spawns for special roles
- Test spawns after setting them

### ❌ DON'T

- Don't forget to set default spawns
- Don't place spawns in walls or hazards
- Don't overlap faction spawns (causes confusion)

## See Also

- [Spawnsaver Plugin](spawnsaver.md) - Saves player position on disconnect
- [Factions System](../systems/factions.md) - Faction creation
- [Classes System](../systems/classes.md) - Class creation
- Source: `plugins/spawns.lua`
