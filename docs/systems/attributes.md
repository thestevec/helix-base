# Attributes System

> **Reference**: `gamemode/core/libs/sh_attribs.lua`, `gamemode/core/meta/sh_character.lua`

The Helix attributes system provides RPG-style character progression through customizable stats that can affect gameplay.

## ⚠️ Important: Use Built-in Helix Functions

**Always use Helix's built-in attribute functions** rather than creating custom stat systems. The framework provides:
- Automatic database persistence
- Network synchronization
- Temporary boosts system
- Max value capping
- Setup hooks for gameplay effects
- Integration with character system

## Core Concepts

### What are Attributes?

Attributes are numeric character stats (like strength, endurance, intelligence) that:
- Persist in the database
- Can be upgraded with attribute points
- Can have temporary boosts (from items, effects)
- Can trigger gameplay changes via hooks
- Are displayed in character UI

### Built-in Attribute Plugins

Helix includes these attribute plugins by default:
- **Stamina** (`plugins/stamina`) - Adds stamina (stm) and endurance (end) attributes
- **Strength** (`plugins/strength`) - Adds strength (str) attribute

**⚠️ Use these plugins** rather than creating duplicate systems.

## Registering Attributes

### Loading from Directory

**Reference**: `gamemode/core/libs/sh_attribs.lua:11`

**The framework automatically loads attributes** from these directories:
- `schema/attributes/` - Schema-specific attributes
- `plugins/yourplugin/attributes/` - Plugin attributes

```lua
-- File: schema/attributes/sh_intelligence.lua
ATTRIBUTE.name = "Intelligence"
ATTRIBUTE.description = "Affects hacking speed and puzzle solving"
ATTRIBUTE.maxValue = 100  -- Maximum attribute value (default: 100)

-- Called when player spawns or attribute changes
function ATTRIBUTE:OnSetup(client, value)
    -- Apply gameplay effects based on attribute value
    if value >= 50 then
        client:SetRunSpeed(client:GetRunSpeed() * 1.1)
    end
end
```

**File naming convention**: `sh_attributename.lua` where `attributename` is the attribute's unique ID.

### Attribute Structure

```lua
ATTRIBUTE.name = "Attribute Name"           -- Display name (required)
ATTRIBUTE.description = "What it does"      -- Description (required)
ATTRIBUTE.maxValue = 100                    -- Max value (default: 100)

-- Optional: Called when attribute is set up or changed
function ATTRIBUTE:OnSetup(client, value)
    -- Apply effects to player based on attribute value
end
```

## Using Character Attributes

### Getting Attribute Values

**Reference**: `gamemode/core/meta/sh_character.lua`

```lua
-- Get attribute value (both CLIENT and SERVER)
local character = client:GetCharacter()
local strength = character:GetAttribute("str", 0)  -- Default to 0 if not set

-- Get all attributes
local attributes = character:GetAttributes()
-- Returns: {str = 5, end = 3, stm = 4, ...}

-- Check in condition
if character:GetAttribute("int") >= 50 then
    -- Player is intelligent enough
end
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't access character data directly
local str = character.attribs["str"]  -- Use GetAttribute() instead
```

### Setting Attribute Values

**Reference**: `gamemode/core/libs/sh_attribs.lua:85`

```lua
-- SERVER ONLY
-- Set attribute to specific value
character:SetAttrib("str", 10)

-- Framework automatically:
-- - Caps at maxValue
-- - Saves to database
-- - Syncs to client
-- - Calls OnSetup hook
-- - Fires CharacterAttributeUpdated hook
```

### Updating (Adding to) Attributes

**Reference**: `gamemode/core/libs/sh_attribs.lua:54`

```lua
-- SERVER ONLY
-- Add to existing attribute value
character:UpdateAttrib("str", 2)  -- Add 2 to strength

-- Example: Leveling up
function PLUGIN:PlayerLevelUp(client)
    local character = client:GetCharacter()
    character:UpdateAttrib("str", 1)
    character:UpdateAttrib("end", 1)
end

-- Framework automatically caps at max value
-- If str is 98 and max is 100, adding 5 results in 100
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't manually modify attributes
local attribs = character:GetAttributes()
attribs["str"] = attribs["str"] + 2
character:SetAttributes(attribs)
-- Use UpdateAttrib() instead!
```

## Attribute Boosts

### Temporary Boosts

**Reference**: `gamemode/core/libs/sh_attribs.lua:117`

Boosts are temporary attribute increases (from consumables, buffs, equipment):

```lua
-- SERVER ONLY
-- Add temporary boost
character:AddBoost(boostID, attributeID, amount)

-- Example: Strength potion
ITEM.functions.Use = {
    OnRun = function(item)
        local character = item.player:GetCharacter()

        -- Add +5 strength for this boost ID
        character:AddBoost("strengthpotion", "str", 5)

        -- Remove after 60 seconds
        timer.Simple(60, function()
            if character then
                character:RemoveBoost("strengthpotion", "str")
            end
        end)

        return true  -- Consume item
    end
}
```

### Removing Boosts

**Reference**: `gamemode/core/libs/sh_attribs.lua:132`

```lua
-- SERVER ONLY
-- Remove specific boost
character:RemoveBoost(boostID, attributeID)

-- Example: Equipment unequipped
function ITEM:OnUnequipped(client)
    local character = client:GetCharacter()
    character:RemoveBoost(self:GetID(), "str")
end
```

### How Boosts Work

Boosts are stored separately from base attributes:

```lua
-- Character has boosts: {str = {potion = 5, armor = 3}}
-- Base str = 10
-- GetAttribute("str") returns: 10 + 5 + 3 = 18

-- Boosts are NOT saved to database (temporary)
-- Boosts reset on disconnect
-- Multiple boosts stack additively
```

## Attribute Points

### Unspent Points

**Reference**: `gamemode/core/meta/sh_character.lua`

```lua
-- Get unspent attribute points
local points = character:GetAttribPoints()

-- Set attribute points (SERVER)
character:SetAttribPoints(10)

-- Spend point to upgrade attribute
if character:GetAttribPoints() > 0 then
    character:UpdateAttrib("str", 1)
    character:SetAttribPoints(character:GetAttribPoints() - 1)
end
```

### Giving Points on Level Up

```lua
function PLUGIN:CharacterLevelUp(client, character, newLevel)
    -- Give 3 attribute points per level
    local currentPoints = character:GetAttribPoints()
    character:SetAttribPoints(currentPoints + 3)

    client:Notify("You gained 3 attribute points!")
end
```

## Attribute Setup Hook

### OnSetup Function

**Reference**: `gamemode/core/libs/sh_attribs.lua:30`

The `OnSetup` function is called:
- When player spawns with character
- When attribute value changes
- After attribute is updated

```lua
-- File: schema/attributes/sh_stamina.lua
ATTRIBUTE.name = "Stamina"
ATTRIBUTE.description = "Affects how long you can run"
ATTRIBUTE.maxValue = 100

function ATTRIBUTE:OnSetup(client, value)
    -- value = current attribute value

    -- Scale run speed based on stamina
    local baseSpeed = 240
    local bonus = (value / 100) * 50  -- Up to +50 speed at max stamina

    client:SetRunSpeed(baseSpeed + bonus)
end

-- Called automatically by framework when:
-- - Player spawns
-- - Attribute is increased/decreased
-- - Attribute boost is added/removed
```

### Real Example from Stamina Plugin

**Reference**: `plugins/stamina/sh_plugin.lua`

```lua
-- Stamina plugin uses attributes to affect gameplay
PLUGIN.stamina = PLUGIN.stamina or {}

function PLUGIN:PlayerSpawn(client)
    local character = client:GetCharacter()
    if not character then return end

    -- Get endurance attribute
    local endurance = character:GetAttribute("end", 0)

    -- Set max stamina based on endurance
    local maxStamina = 100 + (endurance * 2)  -- +2 stamina per END point
    self.stamina[client] = maxStamina
end
```

## Complete Attribute Example

### Creating Intelligence Attribute

```lua
-- File: schema/attributes/sh_int.lua
ATTRIBUTE.name = "Intelligence"
ATTRIBUTE.description = "Increases hacking speed and unlock ability"
ATTRIBUTE.maxValue = 100

function ATTRIBUTE:OnSetup(client, value)
    -- Faster hacking with higher INT
    client:SetNWInt("hackSpeed", 100 + value)

    -- Unlock high-level hacks at 75+ INT
    if value >= 75 then
        client:SetNWBool("advancedHacker", true)
    else
        client:SetNWBool("advancedHacker", false)
    end
end

-- Use in gameplay
function PLUGIN:PlayerStartHack(client, terminal)
    local character = client:GetCharacter()
    local intelligence = character:GetAttribute("int", 0)

    if intelligence < 25 then
        return false, "Intelligence too low!"
    end

    -- Calculate hack time based on INT
    local baseTime = 30  -- 30 seconds
    local reduction = (intelligence / 100) * 20  -- Up to 20 second reduction
    local hackTime = baseTime - reduction

    -- Start hack minigame with calculated time
    self:StartHackMinigame(client, terminal, hackTime)
end
```

## Attribute Hooks

### CharacterAttributeUpdated

```lua
-- Called when attribute value changes
function PLUGIN:CharacterAttributeUpdated(client, character, key, value)
    -- key = attribute name (e.g., "str")
    -- value = amount changed (can be negative)

    if key == "str" then
        client:ChatPrint("Strength changed by " .. value)
    end
end
```

### CharacterAttributeBoosted

```lua
-- Called when boost is added/removed
function PLUGIN:CharacterAttributeBoosted(client, character, attribID, boostID, removed)
    -- attribID = attribute being boosted (e.g., "str")
    -- boostID = unique boost identifier
    -- removed = true if boost was removed, boost amount otherwise

    if removed then
        client:ChatPrint("Boost expired: " .. boostID)
    else
        client:ChatPrint("Boost applied: +" .. removed .. " to " .. attribID)
    end
end
```

## Configuration

### Max Attribute Value

```lua
-- Default max from config
ix.config.Get("maxAttributes", 100)

-- Or override per attribute
ATTRIBUTE.maxValue = 75  -- This attribute caps at 75
```

## Network Synchronization

**Reference**: `gamemode/core/libs/sh_attribs.lua:48`

Attributes are automatically synchronized:

```lua
-- SERVER sends attribute updates via network
util.AddNetworkString("ixAttributeUpdate")

-- CLIENT receives and updates local character data
net.Receive("ixAttributeUpdate", function()
    local characterID = net.ReadUInt(32)
    local attributeKey = net.ReadString()
    local newValue = net.ReadFloat()

    -- Framework automatically updates character
end)
```

**⚠️ Do NOT** implement custom sync. The framework handles this.

## Best Practices

### ✅ DO

- Use `character:GetAttribute()` to read values
- Use `character:UpdateAttrib()` to change values
- Use `character:AddBoost()` for temporary effects
- Use `ATTRIBUTE:OnSetup()` for gameplay effects
- Store attributes in `schema/attributes/` or `plugins/yourplugin/attributes/`
- Let framework handle database saves
- Use descriptive attribute names ("str", "int", "end")

### ❌ DON'T

- Don't modify `character.attribs` table directly
- Don't create custom attribute systems
- Don't forget to call `OnSetup()` is automatic
- Don't exceed maxValue (framework caps it)
- Don't save boosts to database (they're temporary)
- Don't implement custom network sync

## Common Patterns

### Attribute-Based Requirements

```lua
-- Check attribute before allowing action
function PLUGIN:CanPlayerOpenDoor(client, door)
    local character = client:GetCharacter()
    local strength = character:GetAttribute("str", 0)

    if door:GetClass() == "heavy_door" and strength < 50 then
        client:Notify("You need 50 strength to open this door")
        return false
    end
end
```

### Equipment Attribute Bonuses

```lua
-- Add boost when equipped
function ITEM:OnEquipped(client)
    local character = client:GetCharacter()
    character:AddBoost(self:GetID(), "str", self.strengthBonus or 0)
end

function ITEM:OnUnequipped(client)
    local character = client:GetCharacter()
    character:RemoveBoost(self:GetID(), "str")
end
```

### Scaling with Attributes

```lua
-- Damage scales with strength
function PLUGIN:EntityTakeDamage(target, dmg)
    local attacker = dmg:GetAttacker()

    if IsValid(attacker) and attacker:IsPlayer() then
        local character = attacker:GetCharacter()
        local strength = character:GetAttribute("str", 0)

        -- +1% damage per strength point
        local multiplier = 1 + (strength / 100)
        dmg:ScaleDamage(multiplier)
    end
end
```

## Common Issues

### Attribute not saving

**Cause**: Not using framework functions
**Fix**: Use `character:UpdateAttrib()` or `character:SetAttrib()`

### OnSetup not being called

**Cause**: Function name typo or not defined on ATTRIBUTE table
**Fix**: Ensure `function ATTRIBUTE:OnSetup(client, value)`

### Boost not removing

**Cause**: Incorrect boostID or character invalid
**Fix**: Use same boostID for add/remove, check character validity

### Attributes resetting

**Cause**: Boosts being used for permanent stat changes
**Fix**: Use `UpdateAttrib()` for permanent, `AddBoost()` for temporary

## See Also

- [Character System](character.md) - Character management
- [Stamina Plugin](../plugins/stamina.md) - Example attribute plugin
- [Strength Plugin](../plugins/strength.md) - Example attribute plugin
- Attribute System: `gamemode/core/libs/sh_attribs.lua`
- Character Meta: `gamemode/core/meta/sh_character.lua`
