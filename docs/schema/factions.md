# Creating Factions for Your Schema

> **Reference**: `gamemode/core/libs/sh_faction.lua`, `schema/factions/`

This guide shows you how to create custom factions for your schema. Factions are permanent team assignments that define a character's role, appearance, and starting equipment.

## ⚠️ Important: Use Helix Faction System

**Always create factions in `schema/factions/`** rather than implementing custom team systems. The framework provides:
- Automatic faction registration from files
- Model precaching and validation
- Whitelist management integration
- Character creation UI integration
- Database persistence

## Core Concepts

### What Are Schema Factions?

Factions define the teams in your roleplay setting:
- **Half-Life 2 RP**: Citizens, Civil Protection, Resistance, Vortigaunts
- **Star Wars RP**: Empire, Rebels, Bounty Hunters, Civilians
- **Medieval RP**: Peasants, Knights, Clergy, Merchants

Each faction has:
- Unique player models
- Team color
- Starting equipment
- Special abilities or restrictions
- Optional whitelist requirement

### File-Based Auto-Loading

**Reference**: `gamemode/core/libs/sh_faction.lua:37`

Helix automatically loads faction files from:
```
schema/factions/sh_factionname.lua
```

The file prefix `sh_` makes it shared (loads on both server and client).

## Creating Your First Faction

### Step 1: Create Faction File

Create a new file in `schema/factions/`:

```lua
-- File: schema/factions/sh_citizen.lua
FACTION.name = "Citizen"
FACTION.description = "Regular city residents living under Combine rule"
FACTION.color = Color(50, 150, 50)
FACTION.isDefault = true  -- This is the default faction for new characters

FACTION.models = {
    "models/humans/group01/male_01.mdl",
    "models/humans/group01/male_02.mdl",
    "models/humans/group01/male_03.mdl",
    "models/humans/group01/male_04.mdl",
    "models/humans/group01/male_05.mdl",
    "models/humans/group01/male_06.mdl",
    "models/humans/group01/male_07.mdl",
    "models/humans/group01/male_08.mdl",
    "models/humans/group01/male_09.mdl",
    "models/humans/group01/female_01.mdl",
    "models/humans/group01/female_02.mdl",
    "models/humans/group01/female_03.mdl",
    "models/humans/group01/female_04.mdl",
    "models/humans/group01/female_05.mdl",
    "models/humans/group01/female_06.mdl"
}

-- Store faction index for later reference
FACTION_CITIZEN = FACTION.index
```

### Step 2: Test Your Faction

1. Restart server
2. Create new character
3. Faction should appear in character creation
4. Select faction and choose a model
5. Create character

**⚠️ Important**: The filename determines the faction's unique ID. `sh_citizen.lua` becomes `"citizen"`.

## Faction Properties

### Required Properties

```lua
FACTION.name = "Faction Name"        -- REQUIRED: Display name
FACTION.color = Color(r, g, b)       -- REQUIRED: Team color
FACTION.models = {...}               -- REQUIRED: Player models
```

**Missing any required property will cause an error!**

### Common Properties

```lua
FACTION.description = "Description text"  -- Shown in character creation
FACTION.isDefault = false                -- Make this the default faction
FACTION.flag = "p"                       -- Require 'p' flag to create character
FACTION.pay = 50                         -- Salary amount (if using salary system)
FACTION.weapons = {}                     -- Weapons given on spawn
FACTION.runSpeed = 240                   -- Custom run speed
FACTION.walkSpeed = 100                  -- Custom walk speed
FACTION.jumpPower = 200                  -- Custom jump power
```

## Complete Faction Examples

### Civil Protection Faction

```lua
-- File: schema/factions/sh_police.lua
FACTION.name = "Civil Protection"
FACTION.description = "Combine law enforcement maintaining order in the city"
FACTION.color = Color(25, 25, 112)
FACTION.isDefault = false
FACTION.flag = "p"  -- Requires whitelist

FACTION.models = {
    "models/police.mdl"
}

FACTION.weapons = {
    "weapon_stunstick",
    "weapon_pistol"
}

FACTION.runSpeed = 250
FACTION.walkSpeed = 110

-- Called when character is first created
function FACTION:OnCharacterCreated(client, character)
    -- Give starting items
    local inventory = character:GetInventory()
    inventory:Add("item_radio")
    inventory:Add("item_zipties")

    -- Give starting money
    character:GiveMoney(200)

    -- Set initial rank
    character:SetData("rank", "RECRUIT")
    character:SetData("unit", "UNASSIGNED")
end

-- Called when player spawns
function FACTION:OnSpawn(client)
    -- Set health and armor
    client:SetHealth(150)
    client:SetArmor(50)

    -- Set movement speeds
    client:SetRunSpeed(self.runSpeed)
    client:SetWalkSpeed(self.walkSpeed)

    -- Give weapons
    for _, weapon in ipairs(self.weapons) do
        client:Give(weapon)
    end
end

FACTION_POLICE = FACTION.index
```

### Resistance Faction

```lua
-- File: schema/factions/sh_resistance.lua
FACTION.name = "Resistance"
FACTION.description = "Freedom fighters opposing the Combine occupation"
FACTION.color = Color(150, 50, 50)
FACTION.isDefault = false
FACTION.flag = "r"  -- Requires resistance whitelist

FACTION.models = {
    "models/humans/group03/male_01.mdl",
    "models/humans/group03/male_02.mdl",
    "models/humans/group03/male_03.mdl",
    "models/humans/group03/male_04.mdl",
    "models/humans/group03/male_05.mdl",
    "models/humans/group03/male_06.mdl",
    "models/humans/group03/male_07.mdl",
    "models/humans/group03/male_08.mdl",
    "models/humans/group03/male_09.mdl",
    "models/humans/group03/female_01.mdl",
    "models/humans/group03/female_02.mdl",
    "models/humans/group03/female_03.mdl",
    "models/humans/group03/female_04.mdl",
    "models/humans/group03/female_05.mdl",
    "models/humans/group03/female_06.mdl"
}

function FACTION:OnCharacterCreated(client, character)
    local inventory = character:GetInventory()
    inventory:Add("item_radio_encrypted")
    inventory:Add("item_medkit")
    character:GiveMoney(50)  -- Less money than citizens
end

function FACTION:OnSpawn(client)
    client:SetHealth(100)
    client:SetArmor(25)
end

FACTION_RESISTANCE = FACTION.index
```

### Vortigaunt Faction

```lua
-- File: schema/factions/sh_vortigaunt.lua
FACTION.name = "Vortigaunt"
FACTION.description = "Alien refugees with electrical abilities"
FACTION.color = Color(150, 100, 200)
FACTION.isDefault = false
FACTION.flag = "V"  -- Capital V for special faction

FACTION.models = {
    "models/vortigaunt.mdl",
    "models/vortigaunt_slave.mdl"
}

-- No starting weapons (use powers instead)
FACTION.weapons = {}

function FACTION:OnCharacterCreated(client, character)
    -- Set special data for vortigaunts
    character:SetData("energy", 100)
    character:SetData("enslaved", false)
end

function FACTION:OnSpawn(client)
    -- Vortigaunts have more health
    client:SetHealth(150)
    client:SetArmor(0)

    -- Slightly faster movement
    client:SetRunSpeed(260)
    client:SetWalkSpeed(110)
end

-- Passive energy regeneration
function FACTION:OnThink(client)
    local character = client:GetCharacter()
    if not character then return end

    local energy = character:GetData("energy", 0)
    if energy < 100 then
        character:SetData("energy", math.min(energy + 0.1, 100))
    end
end

FACTION_VORTIGAUNT = FACTION.index
```

## Faction Hooks

### OnCharacterCreated

**Reference**: `gamemode/core/libs/sh_faction.lua` (hook implementation)

Called once when a character is first created with this faction:

```lua
function FACTION:OnCharacterCreated(client, character)
    -- Give starting equipment
    local inventory = character:GetInventory()
    inventory:Add("item_hands")

    -- Set starting money based on faction
    local startMoney = self.pay or 100
    character:GiveMoney(startMoney)

    -- Set faction-specific data
    character:SetData("rank", 1)
    character:SetData("loyaltyPoints", 0)
    character:SetData("joinDate", os.time())

    -- Announce to admins
    for _, ply in ipairs(player.GetAll()) do
        if ply:IsAdmin() then
            ply:ChatPrint(client:Name() .. " created " .. self.name .. " character")
        end
    end
end
```

### OnSpawn

Called every time a player spawns with this faction:

```lua
function FACTION:OnSpawn(client)
    -- Set stats
    client:SetHealth(self.health or 100)
    client:SetArmor(self.armor or 0)

    -- Set speeds
    if self.runSpeed then
        client:SetRunSpeed(self.runSpeed)
    end

    if self.walkSpeed then
        client:SetWalkSpeed(self.walkSpeed)
    end

    -- Give faction weapons
    for _, weapon in ipairs(self.weapons or {}) do
        client:Give(weapon)
    end

    -- Faction-specific spawn effects
    if self.uniqueID == "vortigaunt" then
        client:EmitSound("npc/vortigaunt/vortigaunt_emerge.wav")
    end
end
```

### OnThink

**Reference**: `gamemode/core/libs/sh_faction.lua` (passive effects)

Called every tick for players in this faction:

```lua
function FACTION:OnThink(client)
    local character = client:GetCharacter()
    if not character then return end

    -- Example: Health regeneration for medics
    if self.uniqueID == "medic" then
        if client:Health() < client:GetMaxHealth() then
            client:SetHealth(math.min(client:Health() + 0.05, client:GetMaxHealth()))
        end
    end

    -- Example: Radiation damage for unprotected factions
    if self.uniqueID == "citizen" then
        local radiation = character:GetData("radiation", 0)
        if radiation > 50 then
            client:TakeDamage(0.1, client, client)
        end
    end
end
```

### GetModels

**Reference**: `gamemode/core/libs/sh_faction.lua:71`

Customize model selection based on player or character data:

```lua
function FACTION:GetModels(client)
    local character = client:GetCharacter()

    -- Example: Different models based on gender
    local gender = character:GetData("gender", "male")
    if gender == "female" then
        return self.femaleModels
    end

    return self.maleModels
end

-- Or based on rank
function FACTION:GetModels(client)
    local character = client:GetCharacter()
    local rank = character:GetData("rank", 1)

    if rank >= 5 then
        return self.officerModels
    elseif rank >= 3 then
        return self.veteranModels
    else
        return self.recruitModels
    end
end
```

## Advanced Faction Features

### Model Bodygroups

Specify model with bodygroups:

```lua
FACTION.models = {
    "models/player/group01/male_01.mdl",

    -- Model with skin and bodygroups
    {
        "models/player/group01/male_02.mdl",
        1,          -- Skin index
        "0120000"   -- Bodygroups (as string)
    }
}
```

### Faction-Specific Items

Restrict items to specific factions:

```lua
-- File: schema/items/sh_cp_armor.lua
ITEM.name = "Civil Protection Armor"
ITEM.factionID = FACTION_POLICE  -- Only this faction can use

-- Or check in item hooks
function ITEM:CanTransfer(oldInv, newInv)
    local owner = newInv:GetOwner()
    if not owner then return true end

    local character = owner:GetCharacter()
    if character:GetFaction() != FACTION_POLICE then
        return false
    end

    return true
end
```

### Faction Restrictions

Check faction in hooks:

```lua
-- File: schema/sv_schema.lua
function Schema:CanPlayerUseCharacter(client, character)
    local faction = character:GetFaction()

    -- Limit faction to specific user groups
    if faction == FACTION_ADMIN then
        if not client:IsAdmin() then
            return false, "You must be admin to use this faction"
        end
    end

    return true
end
```

## Best Practices

### ✅ DO

- Create one faction file per faction in `schema/factions/`
- Use `sh_` prefix for automatic loading
- Always set name, color, and models
- Store `FACTION.index` in global variable (e.g., `FACTION_POLICE`)
- Use OnCharacterCreated for one-time setup
- Use OnSpawn for equipment and stats
- Provide meaningful descriptions
- Test all models in-game
- Use flags for whitelisted factions

### ❌ DON'T

- Don't modify Helix core faction system
- Don't create factions without required properties
- Don't forget to set isDefault on main faction
- Don't use same models across all factions
- Don't hardcode faction indices (use stored globals)
- Don't create duplicate faction files
- Don't forget to precache custom models
- Don't bypass framework's faction system

## Common Patterns

### Faction Pay System

```lua
-- Set faction salary
FACTION.pay = 100

-- Use in salary timer (schema/sv_schema.lua)
timer.Create("SalaryPayment", 300, 0, function()
    for _, client in ipairs(player.GetAll()) do
        local character = client:GetCharacter()
        if character then
            local faction = ix.faction.indices[character:GetFaction()]
            if faction and faction.pay > 0 then
                character:GiveMoney(faction.pay)
                client:Notify("Salary: $" .. faction.pay)
            end
        end
    end
end)
```

### Faction Abilities

```lua
-- File: schema/sh_schema.lua
function Schema:KeyPress(client, key)
    if key == IN_RELOAD then
        local character = client:GetCharacter()
        if not character then return end

        local faction = ix.faction.indices[character:GetFaction()]

        -- Vortigaunt zap ability
        if faction.uniqueID == "vortigaunt" then
            self:VortigauntZap(client)
        end
    end
end
```

### Dynamic Faction Names

```lua
function FACTION:GetName(character)
    -- Show rank in faction name
    local rank = character:GetData("rank", "Recruit")
    return rank .. " (" .. self.name .. ")"
end
```

## Testing Your Factions

1. **Create Test Character**: Create character in each faction
2. **Check Models**: Verify all models load correctly
3. **Test Spawning**: Ensure OnSpawn equipment works
4. **Verify Whitelist**: Test flag requirements
5. **Check Colors**: Verify team colors display correctly
6. **Test Hooks**: Confirm OnCharacterCreated runs once

## See Also

- [Faction System](../systems/factions.md) - Core faction system reference
- [Classes](classes.md) - Creating classes within factions
- [Schema Structure](structure.md) - Schema directory layout
- [Character System](../systems/character.md) - Character faction assignment
- Source: `gamemode/core/libs/sh_faction.lua`
