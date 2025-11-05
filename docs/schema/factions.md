# Creating Schema Factions

> **Reference**: `gamemode/core/libs/sh_faction.lua`, `schema/factions/`

Factions are teams that define player roles in your schema. This guide shows how to create factions specifically for your schema.

## ⚠️ Important: Use Helix Faction System

**Always use Helix's built-in faction registration** rather than creating custom team systems. The framework provides:
- Automatic registration from `schema/factions/` folder
- Team index assignment
- Model precaching
- Whitelist integration
- Character creation integration

## Core Concepts

### What are Schema Factions?

Schema factions define the teams available in your roleplay setting:
- **HL2RP**: Citizens, Civil Protection, Resistance, Vortigaunts
- **DarkRP**: Citizens, Police, Mayor, Gang Members
- **Medieval RP**: Peasants, Guards, Nobility, Merchants

Each faction has unique:
- Player models
- Starting equipment
- Permissions and flags
- Spawn behavior

## Creating Factions

### File Location

**Place faction files** in:
```
schema/factions/sh_factionname.lua
```

**File naming convention**:
- `sh_` prefix (shared realm)
- Lowercase faction name
- `.lua` extension

Examples:
- `sh_citizen.lua` → Faction uniqueID: "citizen"
- `sh_police.lua` → Faction uniqueID: "police"
- `sh_resistance.lua` → Faction uniqueID: "resistance"

### Basic Faction Template

```lua
-- schema/factions/sh_citizen.lua
FACTION.name = "Citizen"                          -- Display name
FACTION.description = "Ordinary city residents"   -- Description
FACTION.color = Color(80, 150, 80)                -- Team color
FACTION.isDefault = true                          -- Default faction

FACTION.models = {
    "models/player/group01/male_01.mdl",
    "models/player/group01/male_02.mdl",
    "models/player/group01/male_03.mdl",
    "models/player/group01/male_04.mdl",
    "models/player/group01/male_05.mdl",
    "models/player/group01/female_01.mdl",
    "models/player/group01/female_02.mdl",
    "models/player/group01/female_03.mdl"
}

-- Store faction index for later use
FACTION_CITIZEN = FACTION.index
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't use Garry's Mod team functions
team.SetUp(1, "Citizen", Color(80, 150, 80))

-- WRONG: Don't create custom tables
CUSTOM_FACTIONS = {}
CUSTOM_FACTIONS["citizen"] = {...}
```

## Required Properties

### Minimal Requirements

```lua
FACTION.name = "Display Name"        -- REQUIRED
FACTION.color = Color(r, g, b)       -- REQUIRED
FACTION.models = {...}               -- REQUIRED (or defaults used)
```

**⚠️ If missing**: Framework will show error and faction may not work properly.

## Common Faction Examples

### HL2RP Citizens

```lua
-- schema/factions/sh_citizen.lua
FACTION.name = "Citizen"
FACTION.description = "Loyal city residents living under the Combine"
FACTION.color = Color(80, 120, 180)
FACTION.isDefault = true
FACTION.pay = 25

FACTION.models = {
    "models/player/group01/male_01.mdl",
    "models/player/group01/male_02.mdl",
    "models/player/group01/male_03.mdl",
    "models/player/group01/male_04.mdl",
    "models/player/group01/male_05.mdl",
    "models/player/group01/male_06.mdl",
    "models/player/group01/male_07.mdl",
    "models/player/group01/male_08.mdl",
    "models/player/group01/male_09.mdl",
    "models/player/group01/female_01.mdl",
    "models/player/group01/female_02.mdl",
    "models/player/group01/female_03.mdl",
    "models/player/group01/female_04.mdl",
    "models/player/group01/female_06.mdl"
}

function FACTION:OnCharacterCreated(client, character)
    local inventory = character:GetInventory()
    inventory:Add("cid")  -- Citizen ID card
end

FACTION_CITIZEN = FACTION.index
```

### HL2RP Civil Protection

```lua
-- schema/factions/sh_cp.lua
FACTION.name = "Civil Protection"
FACTION.description = "Enforcers of Combine law and order"
FACTION.color = Color(50, 100, 150)
FACTION.isDefault = false
FACTION.flag = "p"  -- Requires police flag
FACTION.pay = 75

FACTION.models = {
    "models/police.mdl"
}

FACTION.weapons = {
    "weapon_stunstick",
    "weapon_pistol"
}

function FACTION:OnCharacterCreated(client, character)
    local inventory = character:GetInventory()
    inventory:Add("cid_police")
    inventory:Add("handheld_radio")
    character:GiveMoney(100)
end

function FACTION:OnSpawn(client)
    client:SetHealth(125)
    client:SetArmor(50)
    client:SetRunSpeed(250)

    for _, weapon in ipairs(self.weapons) do
        client:Give(weapon)
    end
end

FACTION_CP = FACTION.index
```

### Resistance Fighter

```lua
-- schema/factions/sh_resistance.lua
FACTION.name = "Resistance"
FACTION.description = "Fighters against Combine oppression"
FACTION.color = Color(150, 50, 50)
FACTION.isDefault = false
FACTION.flag = "r"  -- Requires resistance flag

FACTION.models = {
    "models/player/group03/male_01.mdl",
    "models/player/group03/male_02.mdl",
    "models/player/group03/male_03.mdl",
    "models/player/group03/male_04.mdl",
    "models/player/group03/female_01.mdl",
    "models/player/group03/female_02.mdl",
    "models/player/group03/female_03.mdl",
    "models/player/group03/female_04.mdl"
}

FACTION.weapons = {
    "weapon_pistol",
    "weapon_crowbar"
}

function FACTION:OnCharacterCreated(client, character)
    local inventory = character:GetInventory()
    inventory:Add("handheld_radio")
    inventory:Add("item_healthkit")
    character:GiveMoney(50)
end

function FACTION:OnSpawn(client)
    client:SetHealth(100)
    client:SetRunSpeed(260)

    for _, weapon in ipairs(self.weapons) do
        client:Give(weapon)
    end
end

function FACTION:OnThink(client)
    -- Passive stamina regeneration
    local character = client:GetCharacter()
    if character then
        local stamina = character:GetData("stamina", 100)
        if stamina < 100 then
            character:SetData("stamina", math.min(stamina + 0.1, 100))
        end
    end
end

FACTION_RESISTANCE = FACTION.index
```

## Faction Properties

### Display Properties

```lua
FACTION.name = "Faction Name"              -- Shown in character creation
FACTION.description = "Description text"   -- Shown in character creation UI
FACTION.color = Color(r, g, b)             -- Team color (scoreboard, chat)
```

### Permission Properties

```lua
FACTION.isDefault = false                  -- Allow without whitelist
FACTION.flag = "f"                         -- Required character flag
```

### Gameplay Properties

```lua
FACTION.pay = 50                          -- Salary amount (if using salary system)
FACTION.weapons = {"weapon_pistol"}       -- Weapons given on spawn
FACTION.runSpeed = 240                    -- Custom run speed
FACTION.walkSpeed = 100                   -- Custom walk speed
FACTION.jumpPower = 200                   -- Custom jump power
```

### Model Properties

```lua
FACTION.models = {
    "models/player/model.mdl",            -- Simple model path

    -- Or with bodygroups/skin
    {
        "models/player/model.mdl",
        0,          -- Skin index
        "0000000"   -- Bodygroups
    }
}
```

## Faction Hooks

### OnCharacterCreated

**Reference**: `gamemode/core/libs/sh_faction.lua:105`

Called when a character is created in this faction:

```lua
function FACTION:OnCharacterCreated(client, character)
    -- Give starting items
    local inventory = character:GetInventory()
    inventory:Add("item_hands")
    inventory:Add("item_cid")

    -- Set starting money
    character:GiveMoney(self.pay or 100)

    -- Set initial data
    character:SetData("rank", 1)
    character:SetData("joinDate", os.time())

    -- Notify player
    client:ChatPrint("Welcome to the " .. self.name)
end
```

### OnSpawn

Called every time a character spawns:

```lua
function FACTION:OnSpawn(client)
    -- Set health/armor
    client:SetHealth(self.health or 100)
    client:SetArmor(self.armor or 0)

    -- Give weapons
    for _, weapon in ipairs(self.weapons or {}) do
        client:Give(weapon)
    end

    -- Set movement speeds
    if self.runSpeed then
        client:SetRunSpeed(self.runSpeed)
    end

    if self.walkSpeed then
        client:SetWalkSpeed(self.walkSpeed)
    end

    if self.jumpPower then
        client:SetJumpPower(self.jumpPower)
    end
end
```

### OnThink

Called every tick for players in this faction:

```lua
function FACTION:OnThink(client)
    -- Faction-specific passive effects

    -- Example: Health regeneration
    if client:Health() < 100 then
        client:SetHealth(math.min(client:Health() + 0.05, 100))
    end

    -- Example: Stamina drain
    if client:KeyDown(IN_SPEED) then
        local char = client:GetCharacter()
        local stamina = char:GetData("stamina", 100)
        char:SetData("stamina", math.max(stamina - 0.2, 0))
    end
end
```

### GetModels

**Reference**: `gamemode/core/libs/sh_faction.lua:71`

Customize model selection based on character data:

```lua
function FACTION:GetModels(client)
    local character = client:GetCharacter()
    local rank = character:GetData("rank", 1)

    -- Different models based on rank
    if rank >= 5 then
        return {"models/player/elite.mdl"}
    elseif rank >= 3 then
        return {"models/player/officer.mdl"}
    end

    return self.models
end
```

## Advanced Faction Features

### Gender-Specific Models

```lua
FACTION.name = "Citizen"
FACTION.description = "City residents"
FACTION.color = Color(80, 150, 80)

FACTION.maleModels = {
    "models/player/group01/male_01.mdl",
    "models/player/group01/male_02.mdl",
    "models/player/group01/male_03.mdl"
}

FACTION.femaleModels = {
    "models/player/group01/female_01.mdl",
    "models/player/group01/female_02.mdl",
    "models/player/group01/female_03.mdl"
}

function FACTION:GetModels(client)
    local character = client:GetCharacter()
    local gender = character:GetData("gender", "male")

    if gender == "female" then
        return self.femaleModels
    end

    return self.maleModels
end

FACTION.models = FACTION.maleModels  -- Default for creation

FACTION_CITIZEN = FACTION.index
```

### Rank-Based Equipment

```lua
function FACTION:OnSpawn(client)
    local character = client:GetCharacter()
    local rank = character:GetData("rank", 1)

    -- Base equipment
    client:Give("weapon_pistol")

    -- Rank-based weapons
    if rank >= 3 then
        client:Give("weapon_smg1")
    end

    if rank >= 5 then
        client:Give("weapon_ar2")
        client:SetArmor(100)
    end

    -- Rank-based health
    local health = 100 + (rank * 10)
    client:SetHealth(math.min(health, 200))
end
```

### Conditional Faction Access

```lua
-- Use with whitelist system
FACTION.name = "VIP Faction"
FACTION.description = "Exclusive faction"
FACTION.color = Color(255, 215, 0)
FACTION.flag = "v"  -- Requires VIP flag

function FACTION:OnCharacterCreated(client, character)
    -- Give VIP perks
    character:GiveMoney(1000)

    local inventory = character:GetInventory()
    inventory:Add("vip_badge")
    inventory:Add("item_premium_spawn")
end
```

## Multiple Faction Schema Example

### Complete faction set for HL2RP

```
schema/factions/
├── sh_citizen.lua         -- Default, everyone can join
├── sh_cp.lua              -- Requires 'p' flag
├── sh_cpunit.lua          -- Requires 'p' flag, elite CP
├── sh_resistance.lua      -- Requires 'r' flag
├── sh_vortigaunt.lua      -- Requires 'v' flag
└── sh_admin.lua           -- Requires 'a' flag, event characters
```

### Citizen (Default)

```lua
-- sh_citizen.lua
FACTION.name = "Citizen"
FACTION.description = "Ordinary city residents"
FACTION.color = Color(80, 120, 180)
FACTION.isDefault = true
FACTION.pay = 25
FACTION.models = {...}  -- 14 citizen models

FACTION_CITIZEN = FACTION.index
```

### Civil Protection (Whitelisted)

```lua
-- sh_cp.lua
FACTION.name = "Civil Protection"
FACTION.description = "Metro Police Force"
FACTION.color = Color(50, 100, 150)
FACTION.flag = "p"
FACTION.pay = 75
FACTION.models = {"models/police.mdl"}
FACTION.weapons = {"weapon_stunstick", "weapon_pistol"}

FACTION_CP = FACTION.index
```

### Resistance (Whitelisted)

```lua
-- sh_resistance.lua
FACTION.name = "Resistance"
FACTION.description = "Anti-Combine fighters"
FACTION.color = Color(150, 50, 50)
FACTION.flag = "r"
FACTION.models = {...}  -- Group03 models
FACTION.weapons = {"weapon_pistol", "weapon_crowbar"}

FACTION_RESISTANCE = FACTION.index
```

## Best Practices

### ✅ DO

- Place faction files in `schema/factions/` folder
- Use `sh_` prefix for faction files
- Always set name, color, and models
- Store faction index in global variable (e.g., `FACTION_CITIZEN`)
- Use `OnCharacterCreated` for one-time setup
- Use `OnSpawn` for every spawn
- Set `isDefault = true` for at least one faction
- Use flags for whitelisted factions
- Precache custom models

### ❌ DON'T

- Don't modify `team.*` functions directly
- Don't create factions outside `schema/factions/`
- Don't forget to set required properties
- Don't use same models for every faction
- Don't give powerful weapons to default faction
- Don't forget to set isDefault for your main faction
- Don't bypass the faction system

## Common Patterns

### Faction-Specific Starting Loadout

```lua
function FACTION:OnCharacterCreated(client, character)
    local inventory = character:GetInventory()

    -- Everyone gets these
    inventory:Add("item_cid")

    -- Faction-specific items
    if self.uniqueID == "cp" then
        inventory:Add("handheld_radio")
        inventory:Add("handcuffs")
        character:GiveMoney(200)
    elseif self.uniqueID == "medic" then
        inventory:Add("medical_bag")
        inventory:Add("item_healthkit")
        inventory:Add("item_healthkit")
        character:GiveMoney(150)
    else
        character:GiveMoney(100)
    end
end
```

### Dynamic Faction Properties

```lua
function FACTION:OnSpawn(client)
    local character = client:GetCharacter()
    local isVIP = client:GetUserGroup() == "vip"

    -- VIP benefits
    if isVIP then
        client:SetHealth(125)
        client:SetArmor(25)
        client:SetRunSpeed(260)
    else
        client:SetHealth(100)
        client:SetRunSpeed(240)
    end
end
```

## Common Issues

### Faction not appearing in character creation

**Cause**: File not in `schema/factions/` or wrong naming
**Fix**: Place file in `schema/factions/sh_name.lua`

### Models not loading

**Cause**: Invalid model paths
**Fix**: Verify model paths are correct and exist

### Whitelist not working

**Cause**: Flag not set or player doesn't have flag
**Fix**: Set `FACTION.flag = "x"` and ensure player has flag

### Faction index is nil

**Cause**: Accessing FACTION.index too early
**Fix**: Use `FACTION_NAME = FACTION.index` at end of faction file

## See Also

- [Faction System](../systems/factions.md) - Detailed faction system reference
- [Class System](../systems/classes.md) - Jobs within factions
- [Flag System](../systems/flags.md) - Permission flags
- [Schema Classes](classes.md) - Creating classes for your factions
- Source: `gamemode/core/libs/sh_faction.lua`
