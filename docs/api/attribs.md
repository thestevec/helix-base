# Attributes API (ix.attributes)

> **Reference**: `gamemode/core/libs/sh_attribs.lua`

The attributes API provides character progression through customizable attributes (stats like strength, endurance, etc.). This system handles attribute registration, value management, temporary boosts, and automatic synchronization.

## ⚠️ Important: Use Built-in Helix Attribute Functions

**Always use Helix's built-in attribute system** rather than creating custom stat systems. The framework automatically provides:
- Database persistence for attribute values
- Client-server network synchronization
- Temporary boost system for items/effects
- Attribute point management
- Maximum value enforcement
- Automatic hook integration

## Core Concepts

### What are Attributes?

Attributes are character statistics that define capabilities (strength, intelligence, endurance, etc.). They:
- Are defined in `attributes/sh_*.lua` files within plugins
- Have minimum and maximum values
- Can be permanently updated or temporarily boosted
- Automatically sync between server and client
- Persist in the database with characters

### Key Terms

**Attribute**: A registered stat (e.g., "str" for strength)
**Boost**: Temporary attribute modifier with unique ID
**Attribute Points**: Currency for increasing attributes

## Library Functions

### ix.attributes.LoadFromDir

**Reference**: `gamemode/core/libs/sh_attribs.lua:11`

```lua
ix.attributes.LoadFromDir(directory)
```

Loads all attribute definitions from a directory. Used internally by the framework.

**Parameters**:
- `directory` (string) - Path to attributes directory

**Example**:
```lua
-- Loaded automatically by framework
ix.attributes.LoadFromDir("helix/plugins/stamina/attributes")
```

### ix.attributes.Setup

**Reference**: `gamemode/core/libs/sh_attribs.lua:30`

```lua
ix.attributes.Setup(client)
```

Initializes attributes for a client's current character. Calls `OnSetup` for each attribute.

**Parameters**:
- `client` (Player) - Player to setup attributes for

**Example**:
```lua
-- Called automatically when character loads
function PLUGIN:PlayerLoadedCharacter(client, character)
    ix.attributes.Setup(client)
end
```

## Character Methods

### character:GetAttribute

**Reference**: `gamemode/core/libs/sh_attribs.lua:179`

**Realm**: Shared (server + owning client only)

```lua
local value = character:GetAttribute(key, default)
```

Returns the current value of an attribute including any active boosts.

**Parameters**:
- `key` (string) - Attribute name (e.g., "str")
- `default` (number) - Value to return if attribute doesn't exist

**Returns**: (number) Current attribute value with boosts applied

**Example**:
```lua
-- Get character's strength
local strength = character:GetAttribute("str", 0)

-- Check if player is strong enough
if character:GetAttribute("str", 0) >= 10 then
    client:Notify("You're strong enough to lift this!")
end
```

### character:UpdateAttrib

**Reference**: `gamemode/core/libs/sh_attribs.lua:54`

**Realm**: Server

```lua
character:UpdateAttrib(key, value)
```

Increments an attribute by the specified amount. Respects maximum value limits.

**Parameters**:
- `key` (string) - Attribute name
- `value` (number) - Amount to add (can be negative)

**Example**:
```lua
-- Add 2 strength on quest completion
character:UpdateAttrib("str", 2)

-- Reduce endurance by 1
character:UpdateAttrib("end", -1)

-- Training system
if character:GetAttribPoints() > 0 then
    character:UpdateAttrib("str", 1)
    character:SetAttribPoints(character:GetAttribPoints() - 1)
    client:Notify("Strength increased!")
end
```

**Triggers Hook**: `CharacterAttributeUpdated(client, character, key, value)`

### character:SetAttrib

**Reference**: `gamemode/core/libs/sh_attribs.lua:85`

**Realm**: Server

```lua
character:SetAttrib(key, value)
```

Sets an attribute to a specific value (not incremental like UpdateAttrib).

**Parameters**:
- `key` (string) - Attribute name
- `value` (number) - New value to set

**Example**:
```lua
-- Reset strength to 10
character:SetAttrib("str", 10)

-- Set attribute from admin command
character:SetAttrib("int", tonumber(args[1]))
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't manually modify attribute tables
character:GetAttributes()["str"] = 10  -- Won't sync or trigger hooks!
```

**Triggers Hook**: `CharacterAttributeUpdated(client, character, key, value)`

### character:AddBoost

**Reference**: `gamemode/core/libs/sh_attribs.lua:117`

**Realm**: Server

```lua
character:AddBoost(boostID, attribID, boostAmount)
```

Temporarily increases an attribute. Useful for consumables, equipment, or temporary effects. Multiple boosts stack.

**Parameters**:
- `boostID` (string) - Unique identifier for this boost (to remove it later)
- `attribID` (string) - Attribute to boost
- `boostAmount` (number) - Amount to add

**Example**:
```lua
-- Strength potion gives +5 STR
character:AddBoost("potion_strength", "str", 5)

-- Equipment bonus
function ITEM:OnEquipped(client)
    local character = client:GetCharacter()
    character:AddBoost(self.uniqueID, "str", 3)
end

-- Multiple boosts stack
character:AddBoost("buff_a", "str", 5)  -- STR +5
character:AddBoost("buff_b", "str", 3)  -- STR +8 total
```

**Triggers Hook**: `CharacterAttributeBoosted(client, character, attribID, boostID, boostAmount)`

### character:RemoveBoost

**Reference**: `gamemode/core/libs/sh_attribs.lua:132`

**Realm**: Server

```lua
character:RemoveBoost(boostID, attribID)
```

Removes a temporary attribute boost.

**Parameters**:
- `boostID` (string) - Unique identifier of boost to remove
- `attribID` (string) - Attribute that was boosted

**Example**:
```lua
-- Remove strength potion effect
character:RemoveBoost("potion_strength", "str")

-- When equipment is unequipped
function ITEM:OnUnequipped(client)
    local character = client:GetCharacter()
    character:RemoveBoost(self.uniqueID, "str")
end
```

**Triggers Hook**: `CharacterAttributeBoosted(client, character, attribID, boostID, true)`

### character:GetBoost

**Reference**: `gamemode/core/libs/sh_attribs.lua:161`

**Realm**: Shared (server + owning client only)

```lua
local boosts = character:GetBoost(attribID)
```

Returns all active boosts for a specific attribute.

**Parameters**:
- `attribID` (string) - Attribute name

**Returns**: (table or nil) Table of boost IDs and values, or nil if no boosts

**Example**:
```lua
-- Check active strength boosts
local strBoosts = character:GetBoost("str")
if strBoosts then
    for id, amount in pairs(strBoosts) do
        print("Boost:", id, "Amount:", amount)
    end
end
```

### character:GetBoosts

**Reference**: `gamemode/core/libs/sh_attribs.lua:170`

**Realm**: Shared (server + owning client only)

```lua
local boosts = character:GetBoosts()
```

Returns all active boosts for this character across all attributes.

**Returns**: (table) Table of all boosts organized by attribute

**Example**:
```lua
-- Display all active buffs
local boosts = character:GetBoosts()
for attrib, boostList in pairs(boosts) do
    for boostID, amount in pairs(boostList) do
        print(attrib, "boosted by", amount, "from", boostID)
    end
end
```

## Defining Attributes

### Attribute File Structure

**Reference**: `plugins/strength/attributes/sh_str.lua`

Create attribute files in `yourplugin/attributes/sh_attributename.lua`:

```lua
-- attributes/sh_str.lua
ATTRIBUTE.name = "Strength"
ATTRIBUTE.description = "A measure of how strong you are."
ATTRIBUTE.maxValue = 100  -- Optional, defaults to config value

-- Optional: Called when attribute is set up for a player
function ATTRIBUTE:OnSetup(client, value)
    -- Initialize systems that depend on this attribute
end
```

**File Naming**: The filename determines the attribute ID:
- `sh_str.lua` → attribute ID is `"str"`
- `sh_intelligence.lua` → attribute ID is `"intelligence"`

## Complete Examples

### Consumable Item with Temporary Boost

```lua
ITEM.name = "Strength Potion"
ITEM.description = "Temporarily increases strength by 5."
ITEM.model = "models/props_junk/PopCan01a.mdl"

function ITEM:OnUse(client)
    local character = client:GetCharacter()

    -- Add temporary boost
    character:AddBoost(self.uniqueID, "str", 5)

    -- Remove boost after 60 seconds
    timer.Simple(60, function()
        if IsValid(client) and character then
            character:RemoveBoost(self.uniqueID, "str")
            client:Notify("Strength potion effect wore off.")
        end
    end)

    client:Notify("You feel stronger! (+5 STR for 60s)")
end
```

### Equipment with Attribute Bonuses

```lua
ITEM.name = "Power Gauntlets"
ITEM.model = "models/weapons/c_arms.mdl"
ITEM.outfitCategory = "hands"

-- Attribute bonuses
ITEM.attribBoosts = {
    ["str"] = 3
}

function ITEM:OnEquipped(client)
    BaseClass.OnEquipped(self, client)

    local character = client:GetCharacter()

    -- Apply all attribute boosts
    for attrib, amount in pairs(self.attribBoosts or {}) do
        character:AddBoost(self.uniqueID, attrib, amount)
    end
end

function ITEM:OnUnequipped(client)
    BaseClass.OnUnequipped(self, client)

    local character = client:GetCharacter()

    -- Remove all attribute boosts
    for attrib, _ in pairs(self.attribBoosts or {}) do
        character:RemoveBoost(self.uniqueID, attrib)
    end
end
```

### Training System

```lua
-- Command to train attributes with points
ix.command.Add("Train", {
    description = "Spend attribute points to increase an attribute.",
    arguments = {
        ix.type.string,  -- attribute name
        ix.type.number   -- amount to train
    },
    OnRun = function(self, client, attribute, amount)
        local character = client:GetCharacter()
        local points = character:GetAttribPoints()

        -- Validate attribute exists
        if not ix.attributes.list[attribute] then
            return "Invalid attribute."
        end

        -- Check if player has enough points
        if points < amount then
            return "You don't have enough attribute points."
        end

        -- Update attribute and deduct points
        character:UpdateAttrib(attribute, amount)
        character:SetAttribPoints(points - amount)

        return "Trained " .. attribute .. " by " .. amount .. "."
    end
})
```

## Best Practices

### ✅ DO

- Use `character:GetAttribute()` to read current values (includes boosts)
- Use `character:UpdateAttrib()` for permanent increases
- Use `character:AddBoost()` for temporary effects
- Give boosts unique IDs (item uniqueID, effect name, etc.)
- Remove boosts when items are unequipped or effects expire
- Check attribute values before performing strength-dependent actions
- Respect the maxValue configured for attributes

### ❌ DON'T

- Don't modify `character:GetAttributes()` table directly
- Don't create custom stat systems - use framework attributes
- Don't forget to remove boosts when they should expire
- Don't use same boost ID for multiple different boosts
- Don't bypass attribute system for custom character stats
- Don't forget realm checks (UpdateAttrib/SetAttrib are server-only)
- Don't assume attributes exist - check `ix.attributes.list[key]` first

## Common Patterns

### Check Attribute Requirement

```lua
-- Strength requirement for action
function PLUGIN:DoSomething(client)
    local character = client:GetCharacter()
    local str = character:GetAttribute("str", 0)

    if str < 15 then
        client:Notify("You need at least 15 strength to do this.")
        return false
    end

    -- Perform action
    return true
end
```

### Gradual Attribute Gain

```lua
-- Gain attributes through actions (like in stamina plugin)
hook.Add("PlayerPostThink", "AttributeGain", function(client)
    if SERVER and client:GetVelocity():Length() > 200 then
        local character = client:GetCharacter()

        -- Small chance to gain endurance while running
        if math.random(1, 1000) == 1 then
            character:UpdateAttrib("end", 0.1)
        end
    end
end)
```

### Timed Boost with Notification

```lua
function ApplyTimedBuff(character, attrib, amount, duration)
    local client = character:GetPlayer()
    local boostID = "buff_" .. attrib .. "_" .. CurTime()

    -- Apply boost
    character:AddBoost(boostID, attrib, amount)
    client:Notify("Gained +" .. amount .. " " .. attrib .. " for " .. duration .. "s")

    -- Remove after duration
    timer.Simple(duration, function()
        if IsValid(client) and character then
            character:RemoveBoost(boostID, attrib)
            client:Notify("Buff expired.")
        end
    end)
end

-- Usage
ApplyTimedBuff(character, "str", 10, 120)  -- +10 STR for 120 seconds
```

## Common Issues

### Boosts Not Stacking

**Cause**: Using the same boost ID for multiple boosts overwrites the previous value.

**Fix**: Use unique boost IDs for each effect:
```lua
-- WRONG: Same ID overwrites
character:AddBoost("strength_buff", "str", 5)
character:AddBoost("strength_buff", "str", 3)  -- Replaces the 5 with 3

-- CORRECT: Different IDs stack
character:AddBoost("potion_str", "str", 5)
character:AddBoost("equipment_str", "str", 3)  -- Now you have +8 total
```

### Boost Persists After Item Removed

**Cause**: Forgetting to remove boost when item is unequipped or sold.

**Fix**: Always remove boosts in cleanup methods:
```lua
function ITEM:OnUnequipped(client)
    local character = client:GetCharacter()
    character:RemoveBoost(self.uniqueID, "str")
end

function ITEM:OnRemoved()
    if self.player then
        local character = self.player:GetCharacter()
        character:RemoveBoost(self.uniqueID, "str")
    end
end
```

### Attributes Not Syncing to Client

**Cause**: Trying to access attributes on client for non-owned characters.

**Fix**: Attributes only sync to server and owning client:
```lua
-- Only works for LOCAL player's character on client
local character = LocalPlayer():GetCharacter()
local str = character:GetAttribute("str", 0)

-- Can't access other players' attributes on client
local otherChar = otherPlayer:GetCharacter()
local str = otherChar:GetAttribute("str", 0)  -- Won't work!
```

## Related Hooks

### CharacterAttributeUpdated

Called when an attribute is updated or set.

```lua
hook.Add("CharacterAttributeUpdated", "MyHook", function(client, character, key, value)
    print(client:Name() .. "'s " .. key .. " changed by " .. value)
end)
```

### CharacterAttributeBoosted

Called when a boost is added or removed.

```lua
hook.Add("CharacterAttributeBoosted", "MyHook", function(client, character, attribID, boostID, isRemoval)
    if isRemoval == true then
        print("Boost removed:", boostID)
    else
        print("Boost added:", boostID, "Amount:", isRemoval)
    end
end)
```

## See Also

- [Character System](../systems/character.md) - Character management and data
- [Attributes Guide](../systems/attributes.md) - User guide for attributes
- [Items API](item.md) - For creating items that modify attributes
- [Config API](config.md) - Configuring maxAttributes setting
- Source: `gamemode/core/libs/sh_attribs.lua`
