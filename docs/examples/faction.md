# Complete Faction Examples

> **Reference**: `gamemode/core/libs/sh_faction.lua`

This document provides complete, working examples of Helix factions from basic to advanced.

## ⚠️ Important: Use Built-in Faction System

**Always use Helix's faction system** rather than creating custom team management. The framework provides:
- Automatic team registration
- Player model precaching
- Whitelist management
- Character creation integration
- Default faction support
- Faction-specific hooks

## Example 1: Basic Faction

The simplest faction definition - just the required fields.

### File Location

**Place in**: `schema/factions/sh_citizen.lua`

The filename determines the faction's unique ID (`sh_citizen.lua` → `"citizen"`).

### Complete Code

```lua
FACTION.name = "Citizen"
FACTION.description = "Regular city residents"
FACTION.color = Color(50, 150, 50)

FACTION.models = {
    "models/player/group01/male_01.mdl",
    "models/player/group01/male_02.mdl",
    "models/player/group01/male_03.mdl",
    "models/player/group01/female_01.mdl",
    "models/player/group01/female_02.mdl",
    "models/player/group01/female_03.mdl"
}

FACTION.isDefault = true  -- This is the default faction for new characters
```

### Key Points

- **Filename**: `sh_citizen.lua` creates faction ID `"citizen"`
- **Required**: `name`, `color`, `models`
- **`isDefault = true`**: Default faction for character creation
- **`models`**: Player models for this faction

## Example 2: Police Faction with Equipment

A faction that gives weapons and items on spawn.

### File Location

**Place in**: `schema/factions/sh_police.lua`

### Complete Code

```lua
FACTION.name = "Civil Protection"
FACTION.description = "Enforcers of city law and order"
FACTION.color = Color(50, 100, 150)

FACTION.models = {
    "models/police.mdl"
}

-- Require flag to create this faction
FACTION.flag = "p"

-- Starting weapons
FACTION.weapons = {
    "weapon_pistol",
    "weapon_stunstick"
}

-- Starting money
FACTION.pay = 50

-- Custom run speed
FACTION.runSpeed = 250

-- Called when character is created
function FACTION:OnCharacterCreated(client, character)
    -- Give starting items
    local inventory = character:GetInventory()

    inventory:Add("item_radio", 1)
    inventory:Add("item_handcuffs", 1)

    -- Give starting money
    character:GiveMoney(100)

    -- Notify player
    client:Notify("You joined Civil Protection!")
end

-- Called when character spawns
function FACTION:OnSpawn(client)
    -- Extra health and armor
    client:SetHealth(125)
    client:SetArmor(50)

    -- Custom color for entity
    client:SetColor(Color(255, 255, 255, 255))
end

-- Store faction index for later use in code
FACTION_POLICE = FACTION.index
```

### Key Points

- **`flag = "p"`**: Players need 'p' flag to create this faction
- **`weapons = {...}`**: Automatically given on spawn
- **`OnCharacterCreated()`**: Run when character is first created
- **`OnSpawn()`**: Run every time character spawns
- **`FACTION_POLICE`**: Store index for use in other code

## Example 3: Medical Faction with Custom Models

A faction with dynamic model selection.

### File Location

**Place in**: `schema/factions/sh_medical.lua`

### Complete Code

```lua
FACTION.name = "Medical Personnel"
FACTION.description = "Doctors and nurses who provide medical care"
FACTION.color = Color(255, 100, 100)

-- Separate male and female models
FACTION.maleModels = {
    "models/player/odessa.mdl",
    "models/player/monk.mdl"
}

FACTION.femaleModels = {
    "models/player/mossman.mdl"
}

-- All models (required for precaching)
FACTION.models = {
    "models/player/odessa.mdl",
    "models/player/monk.mdl",
    "models/player/mossman.mdl"
}

FACTION.flag = "m"  -- Medical flag
FACTION.pay = 75

-- Custom model selection based on gender
function FACTION:GetModels(client, character)
    -- Check character gender
    local gender = character:GetData("gender", "male")

    if gender == "female" then
        return self.femaleModels
    else
        return self.maleModels
    end
end

-- Called when character is created
function FACTION:OnCharacterCreated(client, character)
    local inventory = character:GetInventory()

    -- Give medical items
    inventory:Add("item_medkit", 2)
    inventory:Add("item_bandage", 5)
    inventory:Add("item_radio", 1)

    character:GiveMoney(150)
end

-- Give regeneration ability
function FACTION:OnCharacterCreated(client, character)
    character:SetData("hasRegen", true)
end

function FACTION:Think(client)
    local character = client:GetCharacter()

    if character and character:GetData("hasRegen") then
        -- Regenerate health slowly
        if client:Health() < client:GetMaxHealth() then
            client:SetHealth(math.min(client:Health() + 0.1, client:GetMaxHealth()))
        end
    end
end

FACTION_MEDICAL = FACTION.index
```

### Key Points

- **`GetModels()`**: Return different models based on character data
- **Store models by type**: Separate male/female arrays
- **`Think()`**: Run every tick for this faction
- **Character data**: Store faction-specific data

## Example 4: Rebel Faction with Restrictions

A faction with whitelist and custom restrictions.

### File Location

**Place in**: `schema/factions/sh_rebel.lua`

### Complete Code

```lua
FACTION.name = "Resistance"
FACTION.description = "Freedom fighters resisting oppression"
FACTION.color = Color(150, 50, 50)

FACTION.models = {
    "models/player/Group03/male_01.mdl",
    "models/player/Group03/male_02.mdl",
    "models/player/Group03/male_03.mdl",
    "models/player/Group03/female_01.mdl",
    "models/player/Group03/female_02.mdl"
}

-- Requires whitelist AND flag
FACTION.flag = "r"

-- Starting weapons
FACTION.weapons = {
    "weapon_smg1",
    "weapon_pistol"
}

-- Custom speeds
FACTION.runSpeed = 260
FACTION.walkSpeed = 100
FACTION.jumpPower = 200

-- Limit number of this faction
FACTION.limit = 4

-- Check if player can create this faction
function FACTION:OnCheckAccess(client)
    -- Check if faction is full
    local count = 0

    for _, v in ipairs(player.GetAll()) do
        local char = v:GetCharacter()

        if char and char:GetFaction() == self.index then
            count = count + 1
        end
    end

    if count >= self.limit then
        return false, "Resistance is full (maximum " .. self.limit .. " members)"
    end

    return true
end

-- Called when character is created
function FACTION:OnCharacterCreated(client, character)
    local inventory = character:GetInventory()

    -- Give rebel equipment
    inventory:Add("item_smg_ammo", 3)
    inventory:Add("item_pistol_ammo", 2)
    inventory:Add("item_medkit", 1)
    inventory:Add("item_grenade", 2)

    character:GiveMoney(50)  -- Rebels start poor
end

-- Called when player spawns
function FACTION:OnSpawn(client)
    client:SetHealth(100)
    client:SetArmor(25)

    -- Give ammo
    client:GiveAmmo(90, "SMG1", true)
    client:GiveAmmo(30, "Pistol", true)
end

-- Custom death handling
function FACTION:OnCharacterDeath(client, killer)
    -- Rebels drop their weapons on death
    if IsValid(killer) and killer:IsPlayer() then
        client:ChatPrint("Your weapons were taken!")
    end
end

FACTION_REBEL = FACTION.index
```

### Key Points

- **`limit`**: Maximum faction members
- **`OnCheckAccess()`**: Custom validation for faction creation
- **Return `false, "message"`**: Deny with reason
- **`OnCharacterDeath()`**: Faction-specific death behavior

## Example 5: VIP Faction with Special Abilities

A faction that requires both flag and custom permission.

### File Location

**Place in**: `schema/factions/sh_vip.lua`

### Complete Code

```lua
FACTION.name = "VIP Citizen"
FACTION.description = "Premium citizens with special privileges"
FACTION.color = Color(255, 215, 0)

FACTION.models = {
    "models/player/mossman.mdl",
    "models/player/odessa.mdl",
    "models/player/kleiner.mdl"
}

FACTION.flag = "v"
FACTION.isDefault = false

-- VIP benefits
FACTION.runSpeed = 280
FACTION.walkSpeed = 110
FACTION.jumpPower = 220

-- Higher starting money
FACTION.pay = 100

-- Check VIP status
function FACTION:OnCheckAccess(client)
    -- Check if player has VIP usergroup
    if !client:IsUserGroup("vip") and !client:IsUserGroup("admin") then
        return false, "This faction requires VIP status"
    end

    return true
end

-- Called when character created
function FACTION:OnCharacterCreated(client, character)
    local inventory = character:GetInventory()

    -- VIP starting kit
    inventory:Add("item_medkit", 3)
    inventory:Add("item_backpack", 1)
    inventory:Add("item_radio", 1)
    inventory:Add("item_flashlight", 1)

    character:GiveMoney(500)  -- VIPs start wealthy

    -- Set VIP data
    character:SetData("isVIP", true)
    character:SetData("vipJoinDate", os.time())
end

-- VIP passive income
function FACTION:Think(client)
    local character = client:GetCharacter()

    if !character or !character:GetData("isVIP") then
        return
    end

    -- Give money every 5 minutes
    local lastPay = character:GetData("lastVIPPay", 0)

    if CurTime() - lastPay >= 300 then
        character:GiveMoney(25)
        client:Notify("You received $25 VIP income")
        character:SetData("lastVIPPay", CurTime())
    end
end

-- VIP spawn benefits
function FACTION:OnSpawn(client)
    client:SetHealth(125)
    client:SetArmor(50)

    -- Visual effect
    local effectdata = EffectData()
    effectdata:SetOrigin(client:GetPos())
    util.Effect("Sparks", effectdata)
end

-- Give VIP flag on creation
function FACTION:OnCharacterCreated(client, character)
    -- Ensure they have VIP flag
    if !character:HasFlags("v") then
        character:GiveFlags("v")
    end
end

FACTION_VIP = FACTION.index
```

### Key Points

- **`OnCheckAccess()`**: Check usergroup/VIP status
- **`Think()`**: Passive abilities (income, regen, etc.)
- **Character data**: Store VIP-specific data
- **Visual effects**: Enhanced spawn effects

## Example 6: Faction with Classes

A faction designed to work with the class system.

### File Location

**Place in**: `schema/factions/sh_combine.lua`

### Complete Code

```lua
FACTION.name = "Combine Overwatch"
FACTION.description = "Elite transhuman soldiers"
FACTION.color = Color(200, 50, 50)

FACTION.models = {
    "models/player/combine_soldier.mdl",
    "models/player/combine_soldier_prisonguard.mdl",
    "models/player/combine_super_soldier.mdl"
}

FACTION.flag = "c"

FACTION.weapons = {
    "weapon_ar2",
    "weapon_pistol"
}

FACTION.runSpeed = 240
FACTION.jumpPower = 190

-- This faction uses classes
FACTION.bUseClasses = true

-- Called when character created
function FACTION:OnCharacterCreated(client, character)
    local inventory = character:GetInventory()

    -- Basic equipment
    inventory:Add("item_ar2_ammo", 3)
    inventory:Add("item_radio", 1)

    character:GiveMoney(200)
end

-- Called when spawning
function FACTION:OnSpawn(client)
    local character = client:GetCharacter()

    if !character then return end

    -- Different stats based on class
    local class = character:GetClass()

    if class == CLASS_SOLDIER then
        client:SetHealth(100)
        client:SetArmor(50)
    elseif class == CLASS_ELITE then
        client:SetHealth(150)
        client:SetArmor(100)
    elseif class == CLASS_COMMANDER then
        client:SetHealth(125)
        client:SetArmor(75)
    else
        -- Default
        client:SetHealth(100)
        client:SetArmor(50)
    end

    -- Give appropriate ammo
    client:GiveAmmo(90, "AR2", true)
    client:GiveAmmo(30, "Pistol", true)
end

-- Get models based on class
function FACTION:GetModels(client, character)
    local class = character:GetClass()

    if class == CLASS_SOLDIER then
        return {"models/player/combine_soldier.mdl"}
    elseif class == CLASS_ELITE then
        return {"models/player/combine_super_soldier.mdl"}
    elseif class == CLASS_COMMANDER then
        return {"models/player/combine_soldier_prisonguard.mdl"}
    end

    return self.models
end

FACTION_COMBINE = FACTION.index
```

### Key Points

- **`bUseClasses = true`**: Enable class system
- **`GetModels()`**: Different models per class
- **`OnSpawn()`**: Different stats per class
- Classes defined in separate files

## ⚠️ Do NOT

```lua
-- WRONG: Don't create custom team system
team.SetUp(10, "MyFaction", Color(255, 0, 0))
-- Use FACTION system instead!

-- WRONG: Don't manually set team
client:SetTeam(FACTION_POLICE)
-- Use character:SetFaction() instead!

-- WRONG: Don't forget required fields
FACTION.name = "Test"
-- Missing color and models!

-- WRONG: Don't use client-only functions
function FACTION:OnSpawn(client)
    LocalPlayer():ChatPrint("Spawned")  -- LocalPlayer() doesn't exist on server!
end

-- WRONG: Don't modify player directly in GetModels
function FACTION:GetModels(client, character)
    client:SetModel("models/player/police.mdl")  -- NO! Just return models
    return self.models
end
```

## Best Practices

### ✅ DO

- Use required fields: `name`, `color`, `models`
- Name files: `sh_factionname.lua`
- Store faction index: `FACTION_NAME = FACTION.index`
- Use `OnCharacterCreated()` for one-time setup
- Use `OnSpawn()` for every spawn
- Use `GetModels()` for dynamic model selection
- Validate in `OnCheckAccess()`
- Use character data for faction-specific state
- Precache all models in `FACTION.models`
- Use descriptive faction names and descriptions

### ❌ DON'T

- Don't use GMod's team system directly
- Don't forget to set `isDefault` for at least one faction
- Don't bypass whitelist with `OnCheckAccess()`
- Don't modify player state in `GetModels()`
- Don't forget realm checks in hooks
- Don't give too many starting weapons
- Don't make all factions default
- Don't forget to clean up faction data

## Advanced Tips

### Linking Factions to Classes

```lua
-- In faction file
FACTION.bUseClasses = true

-- In class file (sh_class.lua)
CLASS.faction = FACTION_POLICE
```

### Faction-Specific Commands

```lua
ix.command.Add("PoliceRadio", {
    description = "Send message on police radio",
    OnRun = function(self, client)
        local character = client:GetCharacter()

        if character:GetFaction() != FACTION_POLICE then
            return "Only police can use this"
        end

        -- Radio logic here
    end
})
```

### Faction Pay System

```lua
-- In schema
timer.Create("ixFactionPay", 600, 0, function()
    for _, client in ipairs(player.GetAll()) do
        local character = client:GetCharacter()

        if character then
            local faction = ix.faction.indices[character:GetFaction()]

            if faction and faction.pay then
                character:GiveMoney(faction.pay)
                client:Notify("You received $" .. faction.pay .. " salary")
            end
        end
    end
end)
```

## See Also

- [Faction System](../systems/factions.md) - Complete faction documentation
- [Class System](../systems/classes.md) - Working with classes
- [Flags System](../systems/flags.md) - Character flags
- [Character System](../systems/character.md) - Character functions
- Source: `gamemode/core/libs/sh_faction.lua` - Faction implementation
