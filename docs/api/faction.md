# Faction API (ix.faction)

> **Reference**: `gamemode/core/libs/sh_faction.lua`

The faction API manages team/faction registration, loading, and information retrieval. Factions are the primary groupings for characters in the framework (e.g., Citizens, Police, Resistance).

## ⚠️ Important: Use Built-in Helix Faction System

**Always use Helix's built-in faction system** rather than creating custom teams. The framework automatically provides:
- Automatic faction loading from directories
- Team registration with Garry's Mod's team system
- Model precaching and management
- Whitelist integration
- Color and name management
- Plugin support

## Core Concepts

### What are Factions?

Factions are character groupings that define:
- Available player models
- Team color and name
- Whitelist requirements
- Available classes
- Character restrictions

### Key Terms

**Faction**: A team/group (e.g., FACTION_CITIZEN, FACTION_POLICE)
**Faction Index**: Numeric team ID
**uniqueID**: String identifier for faction
**isDefault**: Whether faction doesn't require whitelist
**Whitelist**: Permission requirement to play as faction

## Library Tables

### ix.faction.indices

**Reference**: `gamemode/core/libs/sh_faction.lua:7`

**Realm**: Shared

```lua
-- Access by index
local faction = ix.faction.indices[1]
print(faction.name, faction.color)

-- Iterate all factions
for index, faction in pairs(ix.faction.indices) do
    print(index, faction.name)
end
```

### ix.faction.teams

**Reference**: `gamemode/core/libs/sh_faction.lua:6`

**Realm**: Shared

```lua
-- Access by uniqueID
local citizen = ix.faction.teams["citizen"]
print(citizen.name)
```

## Library Functions

### ix.faction.Get

**Reference**: `gamemode/core/libs/sh_faction.lua:89`

**Realm**: Shared

```lua
local faction = ix.faction.Get(identifier)
```

Gets faction by index or uniqueID.

**Parameters**:
- `identifier` (number or string) - Faction index or uniqueID

**Returns**: (table) Faction table

**Example**:
```lua
-- By index
local faction = ix.faction.Get(client:Team())
print("Faction:", faction.name)

-- By uniqueID
local police = ix.faction.Get("police")
if police then
    print("Police color:", police.color)
end

-- Character's faction
local char = client:GetCharacter()
local faction = ix.faction.Get(char:GetFaction())
```

### ix.faction.GetIndex

**Reference**: `gamemode/core/libs/sh_faction.lua:97`

**Realm**: Shared

```lua
local index = ix.faction.GetIndex(uniqueID)
```

Gets numeric index from uniqueID.

**Parameters**:
- `uniqueID` (string) - Faction unique ID

**Returns**: (number) Faction index

**Example**:
```lua
local policeIndex = ix.faction.GetIndex("police")
client:SetTeam(policeIndex)

-- Define faction constant
FACTION_POLICE = ix.faction.GetIndex("police")
```

### ix.faction.LoadFromDir

**Reference**: `gamemode/core/libs/sh_faction.lua:37`

**Realm**: Shared (internal)

```lua
ix.faction.LoadFromDir(directory)
```

Loads factions from directory. Called automatically by framework.

### ix.faction.HasWhitelist

**Reference**: `gamemode/core/libs/sh_faction.lua:110`

**Realm**: Client

```lua
local hasWhitelist = ix.faction.HasWhitelist(faction)
```

Checks if player has whitelist for faction.

**Parameters**:
- `faction` (number) - Faction index

**Returns**: (bool) Whether player has access

**Example**:
```lua
if ix.faction.HasWhitelist(FACTION_POLICE) then
    -- Show police character creation
end
```

## Defining Factions

### Faction File Structure

**File Location**: `schema/factions/sh_factionname.lua`

```lua
-- factions/sh_citizen.lua
FACTION.name = "Citizen"
FACTION.description = "Regular city inhabitants"
FACTION.color = Color(100, 150, 100)
FACTION.isDefault = true  -- No whitelist required
FACTION.models = {
    "models/humans/group01/male_01.mdl",
    "models/humans/group01/male_02.mdl",
    "models/humans/group01/female_01.mdl"
}

-- Optional: Custom model getter
function FACTION:GetModels(client)
    -- Return models based on conditions
    return self.models
end

-- Optional: Called when character is set to this faction
function FACTION:OnCharacterCreated(client, character)
    character:SetData("citizenID", math.random(10000, 99999))
end

-- Optional: Model group with names (for character creation)
FACTION.models = {
    {"models/player/group01/male_01.mdl", "Male 01"},
    {"models/player/group01/female_01.mdl", "Female 01"}
}
```

### Faction Fields

```lua
FACTION.name = "Name"  -- REQUIRED
FACTION.description = "Description"
FACTION.color = Color(r, g, b)  -- REQUIRED
FACTION.uniqueID = "id"  -- Auto-set from filename
FACTION.index = 1  -- Auto-set numeric index
FACTION.isDefault = false  -- True = no whitelist needed
FACTION.models = {}  -- List of model paths
FACTION.limit = 0  -- Max players (0 = unlimited)
```

## Complete Examples

### Police Faction

```lua
-- factions/sh_police.lua
FACTION.name = "Metropolitan Police"
FACTION.description = "Enforces law and order"
FACTION.color = Color(50, 100, 200)
FACTION.models = {
    "models/police.mdl",
    "models/police_fem.mdl"
}

function FACTION:OnCharacterCreated(client, character)
    -- Give starting equipment
    local inv = character:GetInventory()
    inv:Add("stunstick")
    inv:Add("pistol")

    -- Set badge number
    character:SetData("badgeNumber", math.random(1000, 9999))
end
```

### Gender-Based Models

```lua
-- factions/sh_citizen.lua
FACTION.name = "Citizen"
FACTION.description = "City inhabitants"
FACTION.color = Color(100, 150, 100)
FACTION.isDefault = true

local maleModels = {
    "models/humans/group01/male_01.mdl",
    "models/humans/group01/male_02.mdl"
}

local femaleModels = {
    "models/humans/group01/female_01.mdl",
    "models/humans/group01/female_02.mdl"
}

function FACTION:GetModels(client, character)
    local gender = character:GetData("gender", "male")
    return gender == "female" and femaleModels or maleModels
end
```

### Faction Constant Definition

```lua
-- schema/sh_schema.lua
function Schema:Initialize()
    -- Define faction constants
    FACTION_CITIZEN = ix.faction.GetIndex("citizen")
    FACTION_POLICE = ix.faction.GetIndex("police")
    FACTION_MEDIC = ix.faction.GetIndex("medic")
end
```

### Faction Whitelist Check

```lua
ix.command.Add("WhitelistPolice", {
    adminOnly = true,
    arguments = ix.type.player,
    OnRun = function(self, client, target)
        local policeIndex = ix.faction.GetIndex("police")

        target:SetWhitelisted(policeIndex, true)
        return target:Name() .. " whitelisted for Police"
    end
})
```

## Best Practices

### ✅ DO

- Define factions in `schema/factions/` directory
- Always set name and color
- Use `isDefault = true` for public factions
- Precache all models (done automatically)
- Use descriptive faction names
- Set reasonable player limits if needed

### ❌ DON'T

- Don't create factions without names/colors
- Don't use team.SetUp() directly - use faction system
- Don't forget to define at least one default faction
- Don't use invalid model paths
- Don't modify ix.faction.indices directly
- Don't bypass whitelist system

## Common Issues

### "Faction X is missing a name"

**Cause**: Missing `FACTION.name` in faction file.

**Fix**:
```lua
FACTION.name = "Faction Name"  -- Required!
```

### Models Not Showing

**Cause**: Invalid model paths or not precached.

**Fix**: Verify paths are correct:
```lua
FACTION.models = {
    "models/player/group01/male_01.mdl"  -- Correct path
}
```

## Related Hooks

```lua
hook.Add("CanPlayerJoinFaction", "MyHook", function(client, faction)
    -- Restrict faction access
    if faction == FACTION_ADMIN and not client:IsSuperAdmin() then
        return false
    end
end)
```

## See Also

- [Class API](class.md) - Classes within factions
- [Character API](character.md) - Character faction methods
- [Flag API](flag.md) - Faction-specific permissions
- Source: `gamemode/core/libs/sh_faction.lua`
