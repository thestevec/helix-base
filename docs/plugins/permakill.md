# Permakill Plugin

> **Reference**: `plugins/permakill.lua`

The Permakill plugin enables permanent character death on your server. When a character dies, they are marked as permakilled and banned from being played again, creating high-stakes roleplay where death has real consequences.

## ⚠️ Important: Use Built-in Permakill System

**Always use the ix.config permakill settings** rather than implementing custom character banning. The framework provides:
- Automatic character banning on death
- Optional world/self damage exemption
- Hook to prevent permakill in specific situations
- Character data persistence
- Automatic enforcement on respawn

## Core Concepts

### What is Permakill?

Permakill is permanent character death where:
- Character is banned when killed
- Character cannot be selected again
- Player must create new character
- Can be enabled/disabled server-wide
- Can exempt world/self damage

## Server Configuration

### permakill

**Reference**: `plugins/permakill.lua:6-8`

- **Type**: Boolean
- **Default**: false
- **Description**: Enable permanent death on the server
- **Console**: `ix_config_permakill 1`

### permakillWorld

**Reference**: `plugins/permakill.lua:10-12`

- **Type**: Boolean
- **Default**: false
- **Description**: Exempt world and self damage from permakill
- **Console**: `ix_config_permakillWorld 1`

**When enabled**:
- Falling damage doesn't permakill
- Drowning doesn't permakill
- Suicide doesn't permakill
- Self-inflicted damage doesn't permakill

## How It Works

### On Death

**Reference**: `plugins/permakill.lua:14-28`

When a player dies:
1. Checks if permakill is enabled
2. Runs `ShouldPermakillCharacter` hook
3. If permakillWorld is enabled, checks if world/self damage
4. If conditions met, marks character as permakilled

### On Respawn

**Reference**: `plugins/permakill.lua:30-37`

When a player spawns:
1. Checks if character is marked permakilled
2. If yes, permanently bans the character
3. Clears the permakill flag
4. Player is kicked to character selection

## Developer Hooks

### ShouldPermakillCharacter

**Reference**: `plugins/permakill.lua:18-20`

```lua
-- Control whether a character should be permakilled
function PLUGIN:ShouldPermakillCharacter(client, character, inflictor, attacker)
    -- Return false to prevent permakill
    -- Return nil/true to allow permakill

    -- Example: Protect newbies
    if character:GetPlayTime() < 3600 then  -- Less than 1 hour
        return false
    end

    -- Example: Protect in safezones
    if client:GetArea() == "safezone" then
        return false
    end

    -- Example: Only permakill if killed by player
    if !attacker:IsPlayer() then
        return false
    end
end
```

## Complete Examples

### Example 1: Grace Period for New Characters

```lua
function SCHEMA:ShouldPermakillCharacter(client, character, inflictor, attacker)
    local playTime = character:GetPlayTime()

    -- 30 minute grace period
    if playTime < 1800 then
        client:Notify("You were spared due to new character protection.")
        return false
    end
end
```

### Example 2: Faction-Specific Permakill

```lua
function PLUGIN:ShouldPermakillCharacter(client, character, inflictor, attacker)
    local faction = character:GetFaction()

    -- Only permakill rebels
    if faction != FACTION_REBEL then
        return false
    end

    return true
end
```

### Example 3: Event Mode Permakill

```lua
-- Only permakill during events
function SCHEMA:ShouldPermakillCharacter(client, character, inflictor, attacker)
    if !SCHEMA.EventMode then
        client:Notify("Permakill is only active during events.")
        return false
    end

    return true
end
```

### Example 4: Permakill with Confirmation

```lua
function SCHEMA:ShouldPermakillCharacter(client, character, inflictor, attacker)
    -- Give player a chance to be revived within 60 seconds
    character:SetData("permakillPending", CurTime() + 60)

    timer.Create("permakill_" .. client:SteamID(), 60, 1, function()
        if IsValid(client) and client:GetCharacter() == character then
            if character:GetData("permakillPending") then
                character:Ban()
                client:Notify("Your character has been permanently killed.")
            end
        end
    end)

    return false  -- Don't permakill immediately
end

-- In medic revive command:
function PLUGIN:RevivePlayer(client)
    local character = client:GetCharacter()

    if character:GetData("permakillPending") then
        character:SetData("permakillPending", nil)
        timer.Remove("permakill_" .. client:SteamID())
        client:Notify("You have been saved from permakill!")
    end
end
```

## Best Practices

### ✅ DO

- Announce permakill status in server rules
- Use ShouldPermakillCharacter for exemptions
- Enable permakillWorld to prevent accidental deaths
- Test permakill thoroughly before enabling
- Provide clear warnings to players
- Log permakill events

### ❌ DON'T

- Don't enable without warning players
- Don't forget to handle edge cases
- Don't use on casual servers
- Don't forget exemptions for special scenarios
- Don't permakill during server issues
- Don't enable without backup system

## Common Patterns

### Pattern 1: Basic Server-Wide Permakill

```lua
-- In server.cfg
ix_config_permakill "1"
ix_config_permakillWorld "1"  -- Prevent fall/drown deaths
```

### Pattern 2: Conditional Permakill

```lua
-- Only permakill if killed by another player
function PLUGIN:ShouldPermakillCharacter(client, character, inflictor, attacker)
    if attacker:IsPlayer() and attacker != client then
        return true
    end

    return false
end
```

### Pattern 3: Warning Before Permakill

```lua
function PLUGIN:PlayerDeath(client, inflictor, attacker)
    local character = client:GetCharacter()

    if ix.config.Get("permakill") and character then
        client:ChatPrint("Warning: Your character has been permanently killed!")
    end
end
```

## Common Issues

### Character Not Being Banned

**Cause**: ShouldPermakillCharacter returning false
**Fix**: Check hook implementations

### World Damage Still Permakilling

**Cause**: permakillWorld not enabled
**Fix**: Enable via `ix_config_permakillWorld 1`

### Permakill Happening After Revive

**Cause**: PlayerSpawn running after revive
**Fix**: Clear permakill flag when reviving

```lua
function PLUGIN:ReviveCharacter(character)
    character:SetData("permakilled", nil)
end
```

## Technical Details

### Death Flow

**Reference**: `plugins/permakill.lua:14-28`

1. `PlayerDeath` hook fires
2. Checks `ix.config.permakill`
3. Runs `ShouldPermakillCharacter` hook
4. If permakillWorld enabled, checks attacker
5. Sets `permakilled` data flag on character

### Spawn Flow

**Reference**: `plugins/permakill.lua:30-37`

1. `PlayerSpawn` hook fires
2. Checks if character has `permakilled` flag
3. If yes, calls `character:Ban()`
4. Clears the flag
5. Player sent to character select

### Data Storage

Permakill status stored in character data:
- Key: `"permakilled"`
- Value: `true` when character should be banned
- Cleared after banning

## See Also

- [Character System](../systems/characters.md) - Character management
- [Data System](../libraries/data.md) - Persistent data storage
- Source: `plugins/permakill.lua`
