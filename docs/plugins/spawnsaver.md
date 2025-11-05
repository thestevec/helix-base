# Spawn Saver Plugin

> **Reference**: `plugins/spawnsaver.lua`

The Spawn Saver plugin saves a character's position and view angles when they disconnect, then restores them when they rejoin. Players spawn exactly where they logged out, maintaining immersion and preventing spawn point teleportation.

## ⚠️ Important: Use Built-in Position Saving

**The plugin automatically saves positions**. The framework provides:
- Automatic position saving on character save/disconnect
- Automatic position restoration on character load
- View angle preservation
- Map change detection
- Observer mode protection

## How It Works

### On Disconnect

**Reference**: `plugins/spawnsaver.lua:6-20`

When character saves:
1. Gets player's current position
2. Gets player's eye angles
3. If in observer mode, uses pre-observer position
4. Saves position + angles + map name to character data

### On Connect

**Reference**: `plugins/spawnsaver.lua:23-42`

When character loads:
1. Gets saved position data
2. Checks if same map
3. If yes, teleports player to saved position
4. Restores eye angles
5. Clears saved position data

## Complete Example

```lua
-- Prevent position saving in certain conditions
function PLUGIN:CharacterPreSave(character)
    local client = character:GetPlayer()

    if IsValid(client) then
        -- Don't save position if in admin sit
        if client:GetNWBool("inAdminSit") then
            character:SetData("pos", nil)
            return
        end
    end
end
```

## Best Practices

### ✅ DO

- Let the plugin work automatically
- Use observer mode detection (built-in)
- Handle map changes (built-in)

### ❌ DON'T

- Don't manually set position data unless necessary
- Don't forget observer mode saves incorrect position (handled automatically)

## Technical Details

- **Data Key**: `"pos"`
- **Data Format**: `{Vector, Angle, string mapname}`
- **Map Check**: Only restores on same map
- **Observer Protection**: Uses pre-observer position if in noclip

## See Also

- [Spawns Plugin](spawns.md) - Faction/class spawn points
- [Observer Plugin](observer.md) - Admin spectator mode
- [Character System](../systems/characters.md) - Character data
- Source: `plugins/spawnsaver.lua`
