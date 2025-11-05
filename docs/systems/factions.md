# Faction System

> **Reference**: `gamemode/core/libs/sh_faction.lua`

The Helix faction system provides team-based gameplay with customizable player models, permissions, and behaviors.

## ⚠️ Important: Use Built-in Helix Functions

**Always use Helix's built-in faction system** rather than creating custom team systems. The framework provides:
- Automatic team registration
- Model precaching
- Whitelist management
- Character creation integration
- Default faction support

## Core Concepts

### What is a Faction?

Factions are teams/groups that characters belong to:
- Citizens, Police, Rebels, etc.
- Each faction has unique models, colors, and flags
- Characters are assigned to one faction
- Factions can have multiple classes (jobs within the faction)

### Faction vs Class

- **Faction**: Permanent team assignment (e.g., "Police Force")
- **Class**: Temporary job within faction (e.g., "Police Chief", "Police Recruit")

```
Faction: Police
├── Class: Police Chief
├── Class: Police Officer
└── Class: Police Recruit

Faction: Medical
├── Class: Doctor
└── Class: Nurse
```

## Creating Factions

### File Location

**Reference**: `gamemode/core/libs/sh_faction.lua:37`

**The framework automatically loads factions** from:
- `schema/factions/` - Your schema factions
- `plugins/yourplugin/factions/` - Plugin factions

### Basic Faction

```lua
-- File: schema/factions/sh_citizen.lua
FACTION.name = "Citizen"                      -- Display name (required)
FACTION.description = "Regular city residents" -- Description
FACTION.color = Color(50, 150, 50)           -- Team color (required)
FACTION.isDefault = true                      -- Default faction for new chars

FACTION.models = {                            -- Player models (required)
    "models/player/group01/male_01.mdl",
    "models/player/group01/female_01.mdl"
}

-- Unique ID is automatically set from filename (sh_citizen.lua = "citizen")
-- Team index is automatically assigned
```

**File naming**: `sh_factionname.lua` where `factionname` is the faction's unique ID.

### Complete Faction Example

```lua
-- File: schema/factions/sh_police.lua
FACTION.name = "Civil Protection"
FACTION.description = "Enforcers of city law"
FACTION.color = Color(50, 100, 150)
FACTION.pay = 50  -- Optional: Salary amount
FACTION.isDefault = false

FACTION.models = {
    "models/police.mdl"
}

-- Optional: Flag requirements
FACTION.flag = "p"  -- Requires 'p' flag to create character

-- Optional: Weapons given on spawn
FACTION.weapons = {
    "weapon_pistol",
    "weapon_stunstick"
}

-- Optional: Custom model selection
function FACTION:GetModels(client)
    -- Can return different models based on player
    return self.models
end

-- Optional: Called when character is created
function FACTION:OnCharacterCreated(client, character)
    -- Give starting items
    local inventory = character:GetInventory()
    inventory:Add("item_radio")

    -- Give starting money
    character:GiveMoney(100)
end

-- Optional: Called when character spawns
function FACTION:OnSpawn(client)
    -- Give equipment
    client:SetHealth(150)
    client:SetArmor(50)
end

-- Optional: Called every tick for this faction
function FACTION:OnThink(client)
    -- Faction-specific behavior
end

-- Store faction index for later use
FACTION_POLICE = FACTION.index
```

## Faction Properties

### Required Properties

```lua
FACTION.name = "Faction Name"        -- Display name (REQUIRED)
FACTION.color = Color(r, g, b)       -- Team color (REQUIRED)
FACTION.models = {...}               -- Player models (REQUIRED)
```

**⚠️ If any required property is missing**, framework will error:
```
ErrorNoHalt("Faction 'factionname' is missing a name...")
```

### Optional Properties

```lua
FACTION.description = "Description"  -- Shown in char creation
FACTION.isDefault = false            -- Default faction for new characters
FACTION.flag = "f"                   -- Required flag to create character
FACTION.pay = 0                      -- Salary amount (if using salary plugin)
FACTION.weapons = {}                 -- Weapons given on spawn
FACTION.runSpeed = 240               -- Custom run speed
FACTION.walkSpeed = 100              -- Custom walk speed
FACTION.jumpPower = 200              -- Custom jump power
```

### Automatic Properties

**Reference**: `gamemode/core/libs/sh_faction.lua:61`

Framework automatically adds:
```lua
FACTION.index = 1                    -- Numeric team index
FACTION.uniqueID = "citizen"         -- From filename
FACTION.plugin = "pluginname"        -- If loaded by plugin
```

## Getting Factions

### Use Built-in Functions

**Reference**: `gamemode/core/libs/sh_faction.lua:89-103`

```lua
-- Get faction by index
local faction = ix.faction.indices[factionIndex]
local faction = ix.faction.Get(factionIndex)

-- Get faction by unique ID
local faction = ix.faction.teams["citizen"]
local faction = ix.faction.Get("citizen")

-- Get faction from character
local character = client:GetCharacter()
local factionIndex = character:GetFaction()
local faction = ix.faction.indices[factionIndex]

-- Access faction properties
print(faction.name)    -- "Citizen"
print(faction.color)   -- Color(50, 150, 50)
print(faction.models)  -- Table of models
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't access tables directly without Get()
local faction = ix.faction.teams[1]  -- Use ix.faction.Get()
```

### Get Faction Index by Unique ID

**Reference**: `gamemode/core/libs/sh_faction.lua:97`

```lua
-- Get numeric index from unique ID
local factionIndex = ix.faction.GetIndex("citizen")

-- Use in character functions
character:SetFaction(factionIndex)
```

## Default Models

**Reference**: `gamemode/core/libs/sh_faction.lua:9`

If no models are specified, faction uses default citizen models:

```lua
-- Framework provides fallback models
local CITIZEN_MODELS = {
    "models/humans/group01/male_01.mdl",
    "models/humans/group01/male_02.mdl",
    -- ... (23 default models)
    "models/humans/group01/female_04.mdl"
}

FACTION.models = FACTION.models or CITIZEN_MODELS
```

**⚠️ Always specify models** for your factions rather than relying on defaults.

## Model Selection

### GetModels Function

**Reference**: `gamemode/core/libs/sh_faction.lua:71`

Customize model selection:

```lua
-- Default implementation (uses faction.models)
function FACTION:GetModels(client)
    return self.models
end

-- Custom: Different models based on rank
function FACTION:GetModels(client)
    local character = client:GetCharacter()
    local rank = character:GetData("rank", 1)

    if rank >= 5 then
        -- High rank gets special models
        return {"models/police_elite.mdl"}
    end

    return self.models
end

-- Custom: Gender-specific models
function FACTION:GetModels(client)
    local character = client:GetCharacter()
    local gender = character:GetData("gender", "male")

    if gender == "female" then
        return self.femaleModels
    end

    return self.maleModels
end
```

### Model Format

Models can be strings or tables:

```lua
FACTION.models = {
    "models/player/group01/male_01.mdl",  -- Simple string

    -- Table format with bodygroups
    {
        "models/player/group01/male_01.mdl",
        0,  -- Skin
        "0000000"  -- Bodygroups
    }
}
```

## Faction Hooks

### OnCharacterCreated

Called after character is created with this faction:

```lua
function FACTION:OnCharacterCreated(client, character)
    -- Give starting items
    local inventory = character:GetInventory()
    inventory:Add("item_hands")

    -- Set starting money
    character:GiveMoney(self.startMoney or 100)

    -- Set faction-specific data
    character:SetData("rank", 1)
    character:SetData("joinDate", os.time())
end
```

### OnSpawn

Called when character spawns:

```lua
function FACTION:OnSpawn(client)
    -- Set health/armor
    client:SetHealth(self.health or 100)
    client:SetArmor(self.armor or 0)

    -- Give weapons
    for _, weapon in ipairs(self.weapons or {}) do
        client:Give(weapon)
    end

    -- Set speeds
    if self.runSpeed then
        client:SetRunSpeed(self.runSpeed)
    end

    if self.walkSpeed then
        client:SetWalkSpeed(self.walkSpeed)
    end
end
```

### OnThink

Called every tick for players in this faction:

```lua
function FACTION:OnThink(client)
    -- Faction-specific passive effects
    local character = client:GetCharacter()

    -- Example: Regenerate health for medics
    if self.uniqueID == "medic" then
        if client:Health() < client:GetMaxHealth() then
            client:SetHealth(math.min(client:Health() + 0.1, client:GetMaxHealth()))
        end
    end
end
```

## Whitelist System

### Checking Whitelist

**Reference**: `gamemode/core/libs/sh_faction.lua:110` (CLIENT)

```lua
-- CLIENT ONLY
local hasWhitelist = ix.faction.HasWhitelist(factionIndex)

if hasWhitelist then
    -- Player can create character in this faction
end

-- Default factions (isDefault = true) return true
```

### Granting Whitelist

Whitelist is stored in player's local data:

```lua
-- Schema/plugin handles whitelist commands
ix.command.Add("GiveWhitelist", {
    adminOnly = true,
    arguments = {
        ix.type.player,
        ix.type.string  -- Faction unique ID
    },
    OnRun = function(self, client, target, factionID)
        local faction = ix.faction.Get(factionID)
        if not faction then
            return "Invalid faction"
        end

        -- Store in player's data
        local data = target:GetData("whitelists", {})
        data[Schema.folder] = data[Schema.folder] or {}
        data[Schema.folder][factionID] = true
        target:SetData("whitelists", data)

        return "Gave " .. faction.name .. " whitelist to " .. target:Name()
    end
})
```

## Complete Faction Examples

### Military Faction

```lua
-- File: schema/factions/sh_military.lua
FACTION.name = "Military"
FACTION.description = "Armed forces personnel"
FACTION.color = Color(100, 120, 100)
FACTION.isDefault = false
FACTION.flag = "m"  -- Requires military flag

FACTION.models = {
    "models/player/urban.mdl",
    "models/player/soldier_stripped.mdl"
}

FACTION.weapons = {
    "weapon_ar2",
    "weapon_pistol"
}

FACTION.health = 150
FACTION.armor = 100
FACTION.runSpeed = 250

function FACTION:OnCharacterCreated(client, character)
    local inventory = character:GetInventory()
    inventory:Add("item_radio")
    inventory:Add("item_ammo_ar2")
    character:GiveMoney(200)
end

function FACTION:OnSpawn(client)
    client:SetHealth(self.health)
    client:SetArmor(self.armor)
    client:SetRunSpeed(self.runSpeed)

    for _, wep in ipairs(self.weapons) do
        client:Give(wep)
    end
end

FACTION_MILITARY = FACTION.index
```

### Medical Faction

```lua
-- File: schema/factions/sh_medic.lua
FACTION.name = "Medical Staff"
FACTION.description = "Doctors and nurses"
FACTION.color = Color(200, 50, 50)
FACTION.isDefault = false

FACTION.models = {
    "models/player/kleiner.mdl",
    "models/player/mossman.mdl"
}

function FACTION:OnCharacterCreated(client, character)
    -- Give medical items
    local inventory = character:GetInventory()
    inventory:Add("item_medkit")
    inventory:Add("item_medkit")
    inventory:Add("item_bandage")
end

function FACTION:OnThink(client)
    -- Passive health regeneration
    if client:Health() < 100 then
        client:SetHealth(math.min(client:Health() + 0.05, 100))
    end
end

function FACTION:GetModels(client)
    local character = client:GetCharacter()

    if character:GetData("gender") == "female" then
        return {"models/player/mossman.mdl"}
    end

    return {"models/player/kleiner.mdl"}
end

FACTION_MEDIC = FACTION.index
```

## Using Faction Indices

Store faction indices for later use:

```lua
-- At end of faction file
FACTION_CITIZEN = FACTION.index

-- Use in schema/plugins
if character:GetFaction() == FACTION_CITIZEN then
    -- Character is a citizen
end

-- Check multiple factions
local civilianFactions = {FACTION_CITIZEN, FACTION_REFUGEE}
if table.HasValue(civilianFactions, character:GetFaction()) then
    -- Character is civilian
end
```

## Best Practices

### ✅ DO

- Store factions in `schema/factions/` folder
- Use `sh_` prefix for faction files
- Always set name, color, and models
- Use `FACTION:OnCharacterCreated()` for starting items
- Use `FACTION:OnSpawn()` for equipment/stats
- Store faction index in global (e.g., `FACTION_POLICE`)
- Use `ix.faction.Get()` to retrieve factions
- Precache any custom models

### ❌ DON'T

- Don't modify `team` Garry's Mod functions directly
- Don't access `ix.faction.indices` directly (use Get())
- Don't create factions without required properties
- Don't forget to set isDefault for main faction
- Don't use same models across all factions
- Don't implement custom team systems

## Common Patterns

### Faction-Specific Commands

```lua
ix.command.Add("FactionAbility", {
    description = "Use faction ability",
    OnRun = function(self, client)
        local character = client:GetCharacter()
        local faction = ix.faction.indices[character:GetFaction()]

        if faction.uniqueID == "police" then
            -- Police ability
            client:ChatPrint("Calling backup!")
        elseif faction.uniqueID == "medic" then
            -- Medic ability
            client:ChatPrint("Healing nearby players")
        end
    end
})
```

### Faction Restrictions

```lua
function PLUGIN:CanPlayerUseItem(client, item)
    local character = client:GetCharacter()
    local faction = ix.faction.indices[character:GetFaction()]

    -- Restrict item to faction
    if item.faction and item.faction != faction.uniqueID then
        client:Notify("Only " .. item.factionName .. " can use this")
        return false
    end
end
```

### Salary Based on Faction

```lua
function PLUGIN:PaySalary()
    for _, client in ipairs(player.GetAll()) do
        local character = client:GetCharacter()
        if character then
            local faction = ix.faction.indices[character:GetFaction()]
            local salary = faction.pay or 0

            if salary > 0 then
                character:GiveMoney(salary)
                client:Notify("Received salary: $" .. salary)
            end
        end
    end
end
```

## Common Issues

### "Faction missing a name"

**Cause**: FACTION.name not set
**Fix**: Add `FACTION.name = "Name"` to faction file

### Models not loading

**Cause**: Models not precached
**Fix**: Framework auto-precaches models in FACTION.models

### Faction not appearing

**Cause**: File not in correct location or wrong naming
**Fix**: Place in `schema/factions/sh_name.lua`

### Whitelist not working

**Cause**: CLIENT-only function used on SERVER
**Fix**: Use appropriate realm for whitelist checks

## See Also

- [Class System](classes.md) - Jobs within factions
- [Character System](character.md) - Character faction assignment
- [Flag System](flags.md) - Permission flags
- Faction System: `gamemode/core/libs/sh_faction.lua`
