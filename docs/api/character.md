# Character API (ix.char)

> **Reference**: `gamemode/core/libs/sh_character.lua`, `gamemode/core/meta/sh_character.lua`

The character API manages character creation, loading, saving, and all character-related data. Characters represent player personas with persistent data including name, description, faction, inventory, money, and attributes.

## ⚠️ Important: Use Built-in Helix Character Functions

**Always use Helix's built-in character system** rather than creating custom player data systems. The framework automatically provides:
- Database persistence and auto-save
- Network synchronization between server and clients
- Character lifecycle management (create, load, save, delete)
- Inventory and money management
- Attribute and flag systems
- Faction and class assignment
- Automatic hook integration

**WARNING**: This API can corrupt character data if misused. For most development, use character:Get/Set methods rather than direct database manipulation.

## Core Concepts

### What is a Character?

A character is a persistent roleplay persona owned by a player. Key points:
- Players can own multiple characters
- Only one character is active at a time
- Characters persist in database across sessions
- All loaded characters are in `ix.char.loaded` table
- Character data syncs automatically between server and clients

### Character Variables

All character data is stored as "variables" registered with `ix.char.RegisterVar`. Each variable automatically gets:
- `Get[VarName]()` accessor method
- `Set[VarName](value)` modifier method (server-only)
- Automatic database persistence
- Network synchronization
- Hook triggers on changes

## Library Tables

### ix.char.loaded

**Reference**: `gamemode/core/libs/sh_character.lua:22`

**Realm**: Shared

Table of all loaded characters indexed by character ID.

```lua
-- Access a character by ID
local character = ix.char.loaded[characterID]

-- Iterate all loaded characters
for id, character in pairs(ix.char.loaded) do
    print(id, character:GetName())
end
```

**Note**: Characters remain loaded after player disconnects. Don't assume character owner is online.

### ix.char.vars

**Reference**: `gamemode/core/libs/sh_character.lua:29`

**Realm**: Shared

Table of registered character variable definitions.

```lua
-- Check if a variable exists
if ix.char.vars["name"] then
    print("Name variable registered")
end
```

### ix.char.cache

**Reference**: `gamemode/core/libs/sh_character.lua:35`

**Realm**: Server

Characters grouped by Steam ID64 of owning players.

```lua
-- Get all character IDs for a player
local steamID = client:SteamID64()
local characterIDs = ix.char.cache[steamID]
```

## Library Functions

### ix.char.Create

**Reference**: `gamemode/core/libs/sh_character.lua:45`

**Realm**: Server

```lua
ix.char.Create(data, callback)
```

Creates a new character and saves it to the database.

**Parameters**:
- `data` (table) - Character properties
  - `name` (string) - Character name
  - `description` (string) - Character description
  - `model` (string) - Character model path
  - `faction` (string) - Faction ID
  - `steamID` (string) - Steam ID64 of owner
  - `money` (number, optional) - Starting money
  - `data` (table, optional) - Custom data table
- `callback` (function, optional) - Called with `callback(characterID)` after creation

**Example**:
```lua
-- Create a new character
local charData = {
    name = "John Doe",
    description = "A mysterious wanderer",
    model = "models/player/group01/male_02.mdl",
    faction = FACTION_CITIZEN,
    steamID = client:SteamID64()
}

ix.char.Create(charData, function(id)
    print("Character created with ID:", id)
    local character = ix.char.loaded[id]
    client:SetCharacter(character)
end)
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't insert into database directly
mysql:Insert("ix_characters"):Execute()  -- Bypasses framework!
```

### ix.char.Restore

**Reference**: `gamemode/core/libs/sh_character.lua:98`

**Realm**: Server

```lua
ix.char.Restore(client, callback, bNoCache, id)
```

Loads a player's characters from database into memory.

**Parameters**:
- `client` (Player) - Player to load characters for
- `callback` (function, optional) - Called with `callback(characterIDs)` when loaded
- `bNoCache` (bool, optional) - Skip cache and force database query
- `id` (number, optional) - Load specific character ID only

**Example**:
```lua
-- Load all player's characters
ix.char.Restore(client, function(characters)
    print("Loaded", #characters, "characters for", client:Name())
end)

-- Load specific character
ix.char.Restore(client, function(chars)
    local character = ix.char.loaded[chars[1]]
    client:SetCharacter(character)
end, false, specificCharID)
```

**Note**: Called automatically when player joins. Rarely needs manual use.

### ix.char.RegisterVar

**Reference**: `gamemode/core/meta/sh_character.lua:264`

**Realm**: Shared

```lua
ix.char.RegisterVar(key, data)
```

Registers a new character variable with auto-generated Get/Set methods.

**Parameters**:
- `key` (string) - Variable name
- `data` (table) - Variable configuration
  - `field` (string, optional) - Database column name
  - `fieldType` (string, optional) - Database field type
  - `default` (any) - Default value
  - `index` (number, optional) - Network order
  - `isLocal` (bool, optional) - Only sync to owner
  - `bNoNetworking` (bool, optional) - Don't sync at all
  - `bNotModifiable` (bool, optional) - No Set method
  - `OnSet` (function, optional) - Custom setter
  - `OnGet` (function, optional) - Custom getter
  - `alias` (string/table, optional) - Alternative names

**Example**:
```lua
-- Register a custom character variable
ix.char.RegisterVar("experience", {
    field = "experience",
    fieldType = "INT(11) UNSIGNED DEFAULT 0",
    default = 0,
    isLocal = true,  -- Only owner sees their XP
    index = 20
})

-- Auto-generates these methods:
-- character:GetExperience(default)
-- character:SetExperience(value)

-- Usage:
local xp = character:GetExperience(0)
character:SetExperience(xp + 100)
```

## Character Methods

### character:GetID

**Reference**: `gamemode/core/meta/sh_character.lua:49`

**Realm**: Shared

```lua
local id = character:GetID()
```

Returns the unique database ID for this character.

**Returns**: (number) Character ID

**Example**:
```lua
local id = character:GetID()
print("Character ID:", id)

-- Use for comparisons
if character:GetID() == 1 then
    print("This is character #1")
end
```

### character:GetPlayer

**Reference**: `gamemode/core/meta/sh_character.lua:236`

**Realm**: Shared

```lua
local client = character:GetPlayer()
```

Returns the player entity that owns this character.

**Returns**: (Player or nil) Owner player if online

**Example**:
```lua
local client = character:GetPlayer()
if IsValid(client) then
    client:Notify("Hello " .. character:GetName())
else
    print("Character owner is offline")
end
```

### character:Save

**Reference**: `gamemode/core/meta/sh_character.lua:61`

**Realm**: Server

```lua
character:Save(callback)
```

Saves character data to database.

**Parameters**:
- `callback` (function, optional) - Called when save completes

**Example**:
```lua
-- Save immediately
character:Save()

-- Save with callback
character:Save(function()
    print("Character saved successfully")
end)
```

**Note**: Framework auto-saves periodically. Manual saves rarely needed.

**Triggers Hooks**:
- `CharacterPreSave(character)` - Before save (return false to cancel)
- `CharacterPostSave(character)` - After save completes

### character:Sync

**Reference**: `gamemode/core/meta/sh_character.lua:100`

**Realm**: Server

```lua
character:Sync(receiver)
```

Synchronizes character data to clients.

**Parameters**:
- `receiver` (Player, optional) - Specific player to sync to. If nil, broadcasts to all.

**Example**:
```lua
-- Sync to all players
character:Sync()

-- Sync to specific player
character:Sync(client)
```

**Note**: Framework handles syncing automatically. Rarely needs manual use.

### character:Setup

**Reference**: `gamemode/core/meta/sh_character.lua:143`

**Realm**: Server

```lua
character:Setup(bNoNetworking)
```

Applies character appearance to player and syncs data.

**Parameters**:
- `bNoNetworking` (bool, optional) - Skip syncing to clients

**Example**:
```lua
-- Full setup with networking
character:Setup()

-- Setup without networking (for testing)
character:Setup(true)
```

**Note**: Called automatically by framework. Manual use rarely needed.

**Triggers Hook**: `CharacterLoaded(character)`

### character:Kick

**Reference**: `gamemode/core/meta/sh_character.lua:194`

**Realm**: Server

```lua
character:Kick()
```

Kicks player off character and returns them to character selection.

**Example**:
```lua
-- Force player back to character menu
character:Kick()

-- Kick on admin action
if not character:HasFlags("admin") then
    character:Kick()
    client:Notify("Character removed by admin")
end
```

### character:Ban

**Reference**: `gamemode/core/meta/sh_character.lua:219`

**Realm**: Server

```lua
character:Ban(time)
```

Bans character from use for specified duration.

**Parameters**:
- `time` (number, optional) - Ban duration in seconds. If nil, permanent ban.

**Example**:
```lua
-- Permanent character ban
character:Ban()

-- 24-hour character ban
character:Ban(86400)

-- Check if banned
if character:GetData("banned") then
    return "Character is banned"
end
```

## Character Variable Methods

All variables registered with `ix.char.RegisterVar` get auto-generated methods. Format: `Get[VarName]()` and `Set[VarName](value)`.

### character:GetName / SetName

**Reference**: `gamemode/core/libs/sh_character.lua:324`

**Realm**: Shared (Get) / Server (Set)

```lua
local name = character:GetName()
character:SetName(newName)  -- Server only
```

Character's display name.

**Example**:
```lua
local name = character:GetName()
client:Notify("Welcome, " .. name)

-- Admin rename
character:SetName("John Smith")
```

### character:GetDescription / SetDescription

**Reference**: `gamemode/core/libs/sh_character.lua:377`

**Realm**: Shared (Get) / Server (Set)

```lua
local desc = character:GetDescription()
character:SetDescription(newDesc)  -- Server only
```

Character's physical description.

**Example**:
```lua
local desc = character:GetDescription()
client:ChatPrint("Description: " .. desc)

-- Update description
character:SetDescription("A tall man with a scar")
```

### character:GetModel / SetModel

**Reference**: `gamemode/core/libs/sh_character.lua:417`

**Realm**: Shared (Get) / Server (Set)

```lua
local model = character:GetModel()
character:SetModel(newModel)  -- Server only
```

Character's player model path.

**Example**:
```lua
local model = character:GetModel()
client:SetModel(model)

-- Change model
character:SetModel("models/player/group01/male_04.mdl")
client:SetModel(character:GetModel())
```

### character:GetClass / SetClass

**Reference**: `gamemode/core/libs/sh_character.lua:530`

**Realm**: Shared (Get) / Server (Set)

```lua
local classID = character:GetClass()
character:SetClass(classID)  -- Server only
```

Character's class ID.

**Example**:
```lua
local classID = character:GetClass()
if classID then
    local class = ix.class.list[classID]
    print("Class:", class.name)
end

-- Assign class
character:SetClass(CLASS_MEDIC)
```

### character:GetFaction / SetFaction

**Reference**: `gamemode/core/libs/sh_character.lua:544`

**Realm**: Shared (Get) / Server (Set)

```lua
local factionID = character:GetFaction()
character:SetFaction(factionID)  -- Server only
```

Character's faction ID.

**Example**:
```lua
local faction = ix.faction.indices[character:GetFaction()]
print("Faction:", faction.name)

-- Change faction
character:SetFaction(FACTION_POLICE)
client:SetTeam(FACTION_POLICE)
```

### character:GetMoney / SetMoney

**Reference**: `gamemode/core/libs/sh_character.lua:687`

**Realm**: Shared (Get, owner only) / Server (Set)

```lua
local money = character:GetMoney()
character:SetMoney(amount)  -- Server only
```

Character's money amount.

**Example**:
```lua
local money = character:GetMoney()
client:Notify("You have " .. ix.currency.Get(money))

-- Give money
character:SetMoney(character:GetMoney() + 500)

-- Take money
if character:GetMoney() >= 100 then
    character:SetMoney(character:GetMoney() - 100)
end
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't modify money directly
character.vars.money = character.vars.money + 100  -- Won't sync!
```

**Triggers Hook**: `CharacterVarChanged(character, "money", oldValue, newValue)`

### character:GetAttributes / SetAttributes

**Reference**: `gamemode/core/libs/sh_character.lua:593`

**Realm**: Shared (Get, owner only) / Server (Set)

```lua
local attribs = character:GetAttributes()
character:SetAttributes(attribTable)  -- Server only
```

Table of all attribute values.

**Example**:
```lua
-- Read attributes
local attribs = character:GetAttributes()
print("STR:", attribs["str"])
print("END:", attribs["end"])

-- Prefer using GetAttribute instead
local str = character:GetAttribute("str", 0)
```

**⚠️ Warning**: Use `character:GetAttribute()` and `character:SetAttrib()` instead. Direct table modification won't trigger updates.

### character:GetData / SetData

**Reference**: `gamemode/core/libs/sh_character.lua:711`

**Realm**: Shared (Get) / Server (Set)

```lua
local value = character:GetData(key, default)
character:SetData(key, value, bNoSave)  -- Server only
```

Generic key-value storage for custom character data.

**Parameters**:
- `key` (string) - Data key
- `value` (any) - Value to store (must be table-serializable)
- `default` (any, optional) - Default if key doesn't exist
- `bNoSave` (bool, optional) - Don't save to database this update

**Example**:
```lua
-- Store custom data
character:SetData("questProgress", {
    quest1 = true,
    quest2 = false
})

-- Retrieve custom data
local quests = character:GetData("questProgress", {})
if quests.quest1 then
    print("Quest 1 completed")
end

-- Temporary data (not saved)
character:SetData("tempBuff", true, true)
```

### character:GetVar / SetVar

**Reference**: `gamemode/core/libs/sh_character.lua:750`

**Realm**: Shared (Get, owner only) / Server (Set)

```lua
local value = character:GetVar(key, default)
character:SetVar(key, value, bNoNetworking, receiver)  -- Server only
```

Temporary character variables (not saved to database).

**Parameters**:
- `key` (string) - Variable key
- `value` (any) - Value to store
- `default` (any, optional) - Default if doesn't exist
- `bNoNetworking` (bool, optional) - Don't sync to clients
- `receiver` (Player, optional) - Sync to specific player only

**Example**:
```lua
-- Temporary combat state
character:SetVar("inCombat", true)

if character:GetVar("inCombat") then
    return "Cannot do this in combat"
end

-- Clear variable
character:SetVar("inCombat", nil)
```

**Note**: Vars are NOT saved to database. Use `GetData/SetData` for persistence.

### character:GetCreateTime

**Reference**: `gamemode/core/libs/sh_character.lua:799`

**Realm**: Shared

```lua
local timestamp = character:GetCreateTime()
```

Unix timestamp when character was created.

**Example**:
```lua
local created = character:GetCreateTime()
local age = os.time() - created
print("Character is", math.floor(age / 86400), "days old")
```

### character:GetLastJoinTime

**Reference**: `gamemode/core/libs/sh_character.lua:811`

**Realm**: Shared

```lua
local timestamp = character:GetLastJoinTime()
```

Unix timestamp of last login with this character.

**Example**:
```lua
local lastJoin = character:GetLastJoinTime()
local diff = os.time() - lastJoin
if diff > 86400 * 7 then
    client:Notify("Welcome back! You were gone for " .. math.floor(diff / 86400) .. " days")
end
```

### character:GetSteamID

**Reference**: `gamemode/core/libs/sh_character.lua:838`

**Realm**: Shared

```lua
local steamID = character:GetSteamID()
```

Steam ID64 of character owner.

**Example**:
```lua
local steamID = character:GetSteamID()
print("Owner Steam ID:", steamID)

-- Find owner even if offline
local owner = player.GetBySteamID64(steamID)
```

## Complete Examples

### Admin Command: Give Money

```lua
ix.command.Add("GiveMoney", {
    description = "Give money to a character.",
    adminOnly = true,
    arguments = {
        ix.type.character,
        ix.type.number
    },
    OnRun = function(self, client, target, amount)
        local character = target:GetCharacter()
        local oldMoney = character:GetMoney()

        character:SetMoney(oldMoney + amount)

        target:Notify("You received " .. ix.currency.Get(amount))
        return "Gave " .. ix.currency.Get(amount) .. " to " .. character:GetName()
    end
})
```

### Quest System

```lua
-- Start quest
function StartQuest(character, questID)
    local quests = character:GetData("quests", {})

    quests[questID] = {
        started = os.time(),
        progress = 0,
        completed = false
    }

    character:SetData("quests", quests)

    local client = character:GetPlayer()
    if IsValid(client) then
        client:Notify("Quest started: " .. questID)
    end
end

-- Check quest status
function HasCompletedQuest(character, questID)
    local quests = character:GetData("quests", {})
    return quests[questID] and quests[questID].completed
end

-- Complete quest
function CompleteQuest(character, questID)
    local quests = character:GetData("quests", {})

    if quests[questID] then
        quests[questID].completed = true
        quests[questID].completedTime = os.time()

        character:SetData("quests", quests)

        -- Reward
        character:SetMoney(character:GetMoney() + 1000)

        local client = character:GetPlayer()
        if IsValid(client) then
            client:Notify("Quest completed! +$1000")
        end
    end
end
```

### Character Inspection System

```lua
ix.command.Add("Examine", {
    description = "Examine another character",
    arguments = ix.type.character,
    OnRun = function(self, client, target)
        local character = target:GetCharacter()

        local info = string.format([[
Name: %s
Description: %s
Faction: %s
Money: %s
Character Age: %d days
        ]],
            character:GetName(),
            character:GetDescription(),
            ix.faction.indices[character:GetFaction()].name,
            ix.currency.Get(character:GetMoney()),
            math.floor((os.time() - character:GetCreateTime()) / 86400)
        )

        client:ChatPrint(info)
    end
})
```

## Best Practices

### ✅ DO

- Use `character:GetPlayer()` and check `IsValid()` before using
- Use `Get/Set` methods for all character data
- Use `GetData/SetData` for custom persistent data
- Use `GetVar/SetVar` for temporary runtime data
- Check if character exists in `ix.char.loaded` before accessing
- Let framework handle saves (auto-saves every few minutes)
- Use character hooks for extending functionality

### ❌ DON'T

- Don't access `character.vars` directly - use Get/Set methods
- Don't create manual database queries for character data
- Don't forget realm restrictions (Set methods are server-only)
- Don't assume `character:GetPlayer()` returns valid player
- Don't modify `ix.char.loaded` directly
- Don't call `Save()` excessively (framework auto-saves)
- Don't store functions or non-serializable data in GetData/SetData

## Common Issues

### Character Data Not Saving

**Cause**: Modifying `character.vars` directly instead of using Set methods.

**Fix**: Always use Set methods:
```lua
-- WRONG
character.vars.money = 1000

-- CORRECT
character:SetMoney(1000)
```

### "Attempt to index nil value" on character

**Cause**: Character not loaded or player offline.

**Fix**: Always validate:
```lua
-- Check character exists
local character = ix.char.loaded[id]
if not character then return end

-- Check player online
local client = character:GetPlayer()
if not IsValid(client) then return end
```

### Data Not Syncing to Client

**Cause**: Variable has `bNoNetworking = true` or `isLocal = true`.

**Fix**: Check variable definition:
```lua
-- Local variables only sync to owner
local money = character:GetMoney()  -- Only owner can see
```

## Related Hooks

### CharacterLoaded

Called when character is loaded and ready to use.

```lua
hook.Add("CharacterLoaded", "MyHook", function(character)
    local client = character:GetPlayer()
    client:Notify("Welcome, " .. character:GetName())
end)
```

### CharacterPreSave / CharacterPostSave

Called before/after character saves.

```lua
hook.Add("CharacterPreSave", "MyHook", function(character)
    -- Modify data before save
    -- Return false to cancel save
end)

hook.Add("CharacterPostSave", "MyHook", function(character)
    print("Character saved:", character:GetName())
end)
```

### CharacterVarChanged

Called when any character variable changes.

```lua
hook.Add("CharacterVarChanged", "MyHook", function(character, key, oldValue, newValue)
    if key == "money" then
        print("Money changed from", oldValue, "to", newValue)
    end
end)
```

## See Also

- [Character System Guide](../systems/character.md) - High-level character concepts
- [Attributes API](attribs.md) - Character attributes and boosts
- [Inventory API](inventory.md) - Character inventory management
- [Faction API](faction.md) - Faction system
- [Class API](class.md) - Class system
- Source: `gamemode/core/libs/sh_character.lua`, `gamemode/core/meta/sh_character.lua`
