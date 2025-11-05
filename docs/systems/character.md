# Character System

The character system is the core of Helix's roleplay functionality. It manages character creation, data storage, attributes, and lifecycle.

## Overview

### What is a Character?

A character in Helix represents a player's in-game persona. Characters have:
- Personal information (name, description, model)
- Faction and class assignments
- Inventory and money
- Attributes and stats
- Custom persistent data

### Character vs Player

- **Player**: The Garry's Mod player entity (persistent across sessions)
- **Character**: A roleplay persona (can have multiple per player)

```
Player (Steam Account)
├── Character 1 (John Doe - Citizen)
├── Character 2 (Jane Smith - Police)
└── Character 3 (Bob Jones - Rebel)
```

## Character Lifecycle

### Character Creation Flow

```
Player Joins
     │
     ▼
┌─────────────────────┐
│  Character Select   │
│  Screen             │
└──────┬──────────────┘
       │
       ├─► Existing Character → Load Character Data
       │                             ↓
       │                        Apply to Player
       │
       └─► Create New
           │
           ▼
    ┌──────────────────┐
    │ Character        │
    │ Creation Panel   │
    │ - Name           │
    │ - Description    │
    │ - Model          │
    │ - Faction        │
    │ - Attributes     │
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │ Validation       │
    │ - Name unique?   │
    │ - Valid faction? │
    │ - Within limit?  │
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │ Save to Database │
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │ Load Character   │
    │ Apply to Player  │
    └──────────────────┘
```

### Character Loading

```lua
-- Server receives character selection from client
function PLUGIN:PlayerLoadedCharacter(client, character, oldCharacter)
    -- Character has been loaded and applied to player
    print(client:Name() .. " loaded character: " .. character:GetName())

    -- Character data is available
    local faction = character:GetFaction()
    local class = character:GetClass()
    local money = character:GetMoney()

    -- Inventory is loaded
    local inventory = character:GetInventory()
end
```

## Character Creation

### Hook into Character Creation

```lua
function PLUGIN:CanPlayerCreateCharacter(client)
    -- Return false to prevent character creation
    if client:GetNWInt("creationCooldown", 0) > CurTime() then
        return false, "You must wait before creating another character"
    end

    return true
end

function PLUGIN:OnCharacterCreated(client, character)
    -- Called after character is created and saved
    print("Character created:", character:GetName())

    -- Give starting items
    local inventory = character:GetInventory()
    inventory:Add("item_hands")
end
```

### Custom Character Fields

```lua
-- Add custom data during creation
function PLUGIN:OnCharacterCreated(client, character)
    character:SetData("playtime", 0)
    character:SetData("karma", 100)
    character:SetData("reputation", {})
end
```

### Character Limits

```lua
-- Limit characters per player
ix.config.Add("maxCharacters", 5, "Maximum characters per player", nil, {
    data = {min = 1, max = 10},
    category = "characters"
})

function PLUGIN:CanPlayerCreateCharacter(client)
    local chars = ix.char.GetAll()
    local count = 0

    for _, char in ipairs(chars) do
        if char:GetPlayer() == client then
            count = count + 1
        end
    end

    if count >= ix.config.Get("maxCharacters", 5) then
        return false, "You have reached the character limit"
    end

    return true
end
```

## Character Properties

### Basic Properties

```lua
local character = client:GetCharacter()

-- Identity
local name = character:GetName()              -- "John Doe"
local desc = character:GetDescription()       -- "A helpful citizen"
local model = character:GetModel()            -- "models/player/..."

-- IDs
local id = character:GetID()                  -- Unique database ID
local steamID = character:GetSteamID()        -- Owner's Steam ID

-- Faction & Class
local factionID = character:GetFaction()      -- Faction ID number
local faction = ix.faction.indices[factionID] -- Faction table
local classID = character:GetClass()          -- Class ID number
local class = ix.class.list[classID]          -- Class table (if assigned)

-- Money
local money = character:GetMoney()            -- Current money amount

-- Inventory
local inventory = character:GetInventory()    -- Inventory object
local invID = character:GetInventoryID()      -- Inventory ID number
```

### Player Association

```lua
-- Get player from character
local player = character:GetPlayer()

-- Get character from player
local character = player:GetCharacter()

-- Check if player has character
if player:GetCharacter() then
    -- Player has active character
end
```

## Character Data

### Setting Data

```lua
-- Set custom data (automatically saved and synced)
character:SetData("job", "Police Officer")
character:SetData("rank", 5)
character:SetData("playtime", 3600)
character:SetData("customData", {
    quests = {"quest1", "quest2"},
    achievements = {},
    reputation = 100
})
```

### Getting Data

```lua
-- Get data with default value
local job = character:GetData("job", "Unemployed")
local rank = character:GetData("rank", 1)
local playtime = character:GetData("playtime", 0)
local customData = character:GetData("customData", {})

-- Check if data exists
if character:GetData("job") then
    -- Character has job data
end
```

### Data Persistence

Character data is automatically:
- Saved to database when changed
- Loaded when character is loaded
- Synchronized to client

### Network Variables

For data that needs to be accessible on client:

```lua
-- SERVER
character:SetNetVar("hunger", 80)

-- CLIENT (automatically synced)
local hunger = character:GetNetVar("hunger", 100)
```

## Character Attributes

### Getting Attributes

```lua
local character = client:GetCharacter()

-- Get attribute value
local strength = character:GetAttribute("str", 0)
local endurance = character:GetAttribute("end", 0)

-- Get all attributes
local attributes = character:GetAttributes()
-- {str = 5, end = 3, stm = 4}
```

### Setting Attributes

```lua
-- Set attribute value
character:SetAttribute("str", 10)

-- Add to attribute
character:UpdateAttrib("str", 2)  -- Adds 2 to current value

-- Set multiple attributes
character:SetAttributes({
    str = 5,
    end = 5,
    stm = 5
})
```

### Attribute Boosts

```lua
-- Temporary boosts (reset on disconnect)
character:AddBoost("str", "powerup", 5)  -- +5 STR from "powerup"

-- Remove boost
character:RemoveBoost("str", "powerup")

-- Get total (base + boosts)
local totalStr = character:GetAttribute("str")
```

### Attribute Points

```lua
-- Get unspent points
local points = character:GetAttribPoints()

-- Set points
character:SetAttribPoints(10)

-- Use point to upgrade attribute
if character:GetAttribPoints() > 0 then
    character:UpdateAttrib("str", 1)
    character:SetAttribPoints(character:GetAttribPoints() - 1)
end
```

## Money System

### Money Operations

```lua
local character = client:GetCharacter()

-- Get money
local money = character:GetMoney()

-- Give money
character:GiveMoney(100)

-- Take money
character:TakeMoney(50)

-- Check if has enough
if character:HasMoney(100) then
    character:TakeMoney(100)
    -- Buy something
end

-- Set money directly
character:SetMoney(500)
```

### Money Transactions

```lua
-- Transfer between characters
local buyer = client:GetCharacter()
local seller = targetClient:GetCharacter()
local price = 100

if buyer:HasMoney(price) then
    buyer:TakeMoney(price)
    seller:GiveMoney(price)
    -- Complete transaction
end
```

### Money Notifications

```lua
-- Notify on money change
function PLUGIN:CharacterMoneyUpdated(character, oldMoney, newMoney)
    local diff = newMoney - oldMoney
    local client = character:GetPlayer()

    if IsValid(client) then
        if diff > 0 then
            client:ChatPrint("You gained $" .. diff)
        elseif diff < 0 then
            client:ChatPrint("You lost $" .. math.abs(diff))
        end
    end
end
```

## Inventory Access

### Getting Inventory

```lua
local character = client:GetCharacter()
local inventory = character:GetInventory()

-- Check if inventory exists
if not inventory then
    print("Character has no inventory")
    return
end

-- Get inventory ID
local invID = inventory:GetID()
```

### Inventory Operations

```lua
-- Add item
inventory:Add("item_medkit")

-- Add at position
inventory:Add("item_medkit", 1, 0, 0)  -- x=0, y=0

-- Check if item can fit
if inventory:CanItemFit(item) then
    inventory:Add(item.uniqueID)
end

-- Get items
local items = inventory:GetItems()
for _, item in pairs(items) do
    print(item:GetName())
end

-- Find item by unique ID
for _, item in pairs(inventory:GetItems()) do
    if item.uniqueID == "item_medkit" then
        -- Found medkit
        break
    end
end
```

## Faction and Class

### Faction Assignment

```lua
-- Get faction
local factionID = character:GetFaction()
local faction = ix.faction.indices[factionID]

print("Faction:", faction.name)
print("Models:", table.concat(faction.models, ", "))
print("Color:", faction.color)

-- Set faction (usually only during creation)
character:SetFaction(factionID)
```

### Class Assignment

```lua
-- Get class
local classID = character:GetClass()
if classID then
    local class = ix.class.list[classID]
    print("Class:", class.name)
end

-- Join class
character:JoinClass(classID)

-- Leave class
character:SetClass()  -- nil = no class
```

### Faction Permissions

```lua
-- Check faction flags
local faction = ix.faction.indices[character:GetFaction()]

-- Check if faction has flag
if faction and faction.HasFlag and faction:HasFlag("v") then
    -- This faction can use vendors
end
```

## Character Flags

### Flag System

Flags are single-character permissions:

```lua
-- Give flag to character
character:GiveFlags("v")   -- Give vendor flag
character:GiveFlags("pm")  -- Give physgun and map flags

-- Take flags
character:TakeFlags("v")

-- Check flags
if character:HasFlags("v") then
    -- Character can use vendors
end

-- Get all flags
local flags = character:GetFlags()  -- "vpm"
```

### Common Flags

```lua
"1" - Default flag (everyone has)
"a" - Access
"m" - Map editing
"p" - Physgun
"t" - Tool gun
"v" - Vendor access
"b" - Business ownership
"d" - Door access
```

## Character Methods

### Utility Methods

```lua
local character = client:GetCharacter()

-- Name operations
character:SetName("New Name")
local name = character:GetName()

-- Description
character:SetDescription("New description")
local desc = character:GetDescription()

-- Model
character:SetModel("models/player/group01/male_02.mdl")
local model = character:GetModel()

-- Bodygroups
character:SetBodygroup(1, 2)
local bodygroup = character:GetBodygroup(1)

-- Skin
character:SetSkin(1)
local skin = character:GetSkin()
```

### Character Validation

```lua
-- Check if character exists
if not IsValid(character) then
    return
end

-- Check if character has player
local player = character:GetPlayer()
if not IsValid(player) then
    print("Character has no player")
    return
end

-- Check if player has character
if not player:GetCharacter() then
    print("Player has no character")
    return
end
```

## Character Deletion

### Deleting Characters

```lua
-- Delete character
character:Delete()

-- Hook for deletion
function PLUGIN:CharacterDeleted(client, character)
    print("Deleted character:", character:GetName())

    -- Clean up character-specific data
    local inventory = character:GetInventory()
    if inventory then
        inventory:Remove()
    end
end
```

### Permakill

```lua
-- Permanent character death
function PLUGIN:PlayerDeath(victim, inflictor, attacker)
    local character = victim:GetCharacter()
    if not character then return end

    if character:GetData("permakill") then
        -- Character is marked for permakill
        timer.Simple(5, function()
            if IsValid(victim) then
                character:Delete()
                victim:KillSilent()
                victim:Spawn()
            end
        end)
    end
end
```

## Character Queries

### Getting All Characters

```lua
-- Get all characters
local allCharacters = ix.char.loaded

for id, character in pairs(allCharacters) do
    print(id, character:GetName())
end
```

### Finding Characters

```lua
-- Find by ID
local character = ix.char.loaded[characterID]

-- Find by player
local character = player:GetCharacter()

-- Find by name
function PLUGIN:FindCharacterByName(name)
    for _, character in pairs(ix.char.loaded) do
        if character:GetName() == name then
            return character
        end
    end
end

-- Find by faction
function PLUGIN:GetCharactersByFaction(factionID)
    local results = {}

    for _, character in pairs(ix.char.loaded) do
        if character:GetFaction() == factionID then
            table.insert(results, character)
        end
    end

    return results
end
```

## Example Usage

### Welcome Message

```lua
function PLUGIN:PlayerLoadedCharacter(client, character, oldChar)
    local name = character:GetName()
    local faction = ix.faction.indices[character:GetFaction()]

    client:ChatPrint("Welcome back, " .. name .. "!")
    client:ChatPrint("You are a " .. faction.name)

    local money = character:GetMoney()
    client:ChatPrint("You have $" .. money)
end
```

### Starting Equipment

```lua
function PLUGIN:OnCharacterCreated(client, character)
    local inventory = character:GetInventory()

    -- Give starting items
    inventory:Add("item_hands")

    -- Give faction-specific items
    local faction = ix.faction.indices[character:GetFaction()]
    if faction.uniqueID == "citizen" then
        inventory:Add("item_phone")
    elseif faction.uniqueID == "police" then
        inventory:Add("item_radio")
        inventory:Add("weapon_pistol")
    end

    -- Give starting money
    character:GiveMoney(100)
end
```

### Playtime Tracking

```lua
function PLUGIN:InitPostEntity()
    timer.Create("TrackPlaytime", 60, 0, function()
        for _, client in ipairs(player.GetAll()) do
            local character = client:GetCharacter()
            if character then
                local playtime = character:GetData("playtime", 0)
                character:SetData("playtime", playtime + 60)
            end
        end
    end)
end

function PLUGIN:PlayerDisconnected(client)
    local character = client:GetCharacter()
    if character then
        -- Save final playtime
        ix.char.Save(character)
    end
end
```

### Character Menu

```lua
-- Command to show character info
ix.command.Add("CharInfo", {
    description = "View character information",
    OnRun = function(self, client)
        local char = client:GetCharacter()
        if not char then return "No character!" end

        local info = {
            "Name: " .. char:GetName(),
            "Description: " .. char:GetDescription(),
            "Faction: " .. ix.faction.indices[char:GetFaction()].name,
            "Money: $" .. char:GetMoney(),
            "Playtime: " .. (char:GetData("playtime", 0) / 60) .. " minutes"
        }

        for _, line in ipairs(info) do
            client:ChatPrint(line)
        end
    end
})
```

## Best Practices

### Always Validate

```lua
-- Bad
local character = client:GetCharacter()
character:GiveMoney(100)  -- Could error if no character!

-- Good
local character = client:GetCharacter()
if not character then return end
character:GiveMoney(100)
```

### Use Data System

```lua
-- Bad: Storing on player entity
client.customData = {}

-- Good: Storing on character
character:SetData("customData", {})
```

### Check Realm

```lua
-- Bad: NetVars on server
if SERVER then
    character:SetNetVar("test", value)  -- NetVars auto-sync
end

-- Good: Regular data on server
if SERVER then
    character:SetData("test", value)
end
```

### Clean Up Data

```lua
function PLUGIN:CharacterDeleted(client, character)
    -- Remove character-specific external data
    self.characterStats[character:GetID()] = nil
end

function PLUGIN:PlayerDisconnected(client)
    local character = client:GetCharacter()
    if character then
        -- Clean up temporary data
        self.playerCache[client:SteamID()] = nil
    end
end
```

## See Also

- [Faction System](factions.md)
- [Class System](classes.md)
- [Inventory System](inventory.md)
- [Attributes System](attributes.md)
- [Character API Reference](../api/character.md)
