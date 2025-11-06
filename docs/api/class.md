# Class API (ix.class)

> **Reference**: `gamemode/core/libs/sh_class.lua`

The class API manages character job assignments within factions. Classes are temporary assignments analogous to "jobs" in a faction - for example, a police faction might have "Police Recruit" and "Police Chief" classes.

## ⚠️ Important: Use Built-in Helix Class System

**Always use Helix's built-in class system** rather than creating custom job systems. The framework automatically provides:
- Class registration and loading from directories
- Automatic class switching and validation
- Player limit enforcement per class
- Hook integration for class changes
- Network synchronization
- Default class fallbacks for factions
- Permission checks via `CanSwitchTo`

## Core Concepts

### What are Classes?

Classes are sub-roles within a faction:
- Characters can switch between classes in their faction
- Each class can have player limits
- Classes can restrict who can join via `CanSwitchTo`
- One class per faction should be marked as `isDefault`
- Classes define OnSet and OnLeave callbacks

### Key Terms

**Class**: A job/role within a faction (e.g., "Police Officer", "Police Chief")
**Default Class**: The fallback class when a character is kicked from their current class
**Class Limit**: Maximum number of players allowed in a class simultaneously
**CanSwitchTo**: Permission function that determines if a player can join a class

## Library Tables

### ix.class.list

**Reference**: `gamemode/core/libs/sh_class.lua:16`

**Realm**: Shared

Array of all registered classes indexed by numeric ID.

```lua
-- Access a class by index
local class = ix.class.list[1]
print("Class Name:", class.name)

-- Iterate all classes
for index, class in ipairs(ix.class.list) do
    print(index, class.name, class.faction)
end
```

## Library Functions

### ix.class.LoadFromDir

**Reference**: `gamemode/core/libs/sh_class.lua:24`

**Realm**: Shared

```lua
ix.class.LoadFromDir(directory)
```

Loads all class files from a directory.

**Parameters**:
- `directory` (string) - Path to class files

**Example**:
```lua
-- Framework loads automatically from schema and plugins
ix.class.LoadFromDir("schema/classes")
```

**Note**: Called automatically by framework. Rarely needs manual use.

### ix.class.Get

**Reference**: `gamemode/core/libs/sh_class.lua:118`

**Realm**: Shared

```lua
local class = ix.class.Get(identifier)
```

Retrieves a class table by its numeric index.

**Parameters**:
- `identifier` (number) - Class index

**Returns**: (table or nil) Class table if found

**Example**:
```lua
-- Get class by ID
local class = ix.class.Get(1)
if class then
    print("Class:", class.name)
    print("Faction:", team.GetName(class.faction))
end

-- Check if class exists
if not ix.class.Get(classID) then
    return "Invalid class"
end
```

### ix.class.CanSwitchTo

**Reference**: `gamemode/core/libs/sh_class.lua:82`

**Realm**: Shared

```lua
local canSwitch, reason = ix.class.CanSwitchTo(client, class)
```

Determines if a player can join a specific class.

**Parameters**:
- `client` (Player) - Player to check
- `class` (number) - Class index to check

**Returns**:
- `canSwitch` (bool) - Whether player can switch
- `reason` (string, optional) - Reason if denied

**Example**:
```lua
-- Check if player can join class
local can, reason = ix.class.CanSwitchTo(client, 2)
if not can then
    client:Notify("Cannot join: " .. (reason or "unknown"))
    return
end

-- Use in command
if ix.class.CanSwitchTo(client, classID) then
    character:JoinClass(classID)
end
```

**Checks Performed**:
- Class exists
- Player's faction matches class faction
- Player not already in that class
- Class not full (if limit set)
- Hook `CanPlayerJoinClass` allows
- Class's custom `CanSwitchTo` allows

### ix.class.GetPlayers

**Reference**: `gamemode/core/libs/sh_class.lua:126`

**Realm**: Shared

```lua
local players = ix.class.GetPlayers(class)
```

Gets all players currently in a specific class.

**Parameters**:
- `class` (number) - Class index

**Returns**: (table) Array of players in the class

**Example**:
```lua
-- Count players in class
local classID = 2
local players = ix.class.GetPlayers(classID)
print("Players in class:", #players)

-- Check if class is full
local class = ix.class.Get(classID)
if class.limit > 0 and #ix.class.GetPlayers(classID) >= class.limit then
    client:Notify("Class is full!")
end

-- Notify all players in a class
for _, ply in ipairs(ix.class.GetPlayers(classID)) do
    ply:Notify("Your class received a message!")
end
```

## Character Methods

### character:JoinClass

**Reference**: `gamemode/core/libs/sh_class.lua:148`

**Realm**: Server

```lua
local success = character:JoinClass(class)
```

Makes a character join a class. Automatically kicks from current class first.

**Parameters**:
- `class` (number) - Class index to join

**Returns**: (bool) Whether successfully joined

**Example**:
```lua
-- Join a class
if character:JoinClass(2) then
    client:Notify("You joined the class!")
else
    client:Notify("Unable to join class")
end

-- Switch classes
local oldClass = character:GetClass()
character:JoinClass(newClassID)
```

**Triggers Hook**: `PlayerJoinedClass(client, class, oldClass)`

### character:KickClass

**Reference**: `gamemode/core/libs/sh_class.lua:169`

**Realm**: Server

```lua
character:KickClass()
```

Kicks character out of their current class and returns them to the default class for their faction.

**Example**:
```lua
-- Kick player from class
character:KickClass()

-- Admin demotion
if not character:HasFlags("admin") then
    character:KickClass()
    client:Notify("You were removed from your class")
end
```

**Note**: Requires a default class (`isDefault = true`) in the faction or will error.

**Triggers Hook**: `PlayerJoinedClass(client, defaultClass)`

## Defining Classes

### Class File Structure

**File Location**: `schema/classes/sh_classname.lua` or `plugins/yourplugin/classes/sh_classname.lua`

**File Naming**: The filename (minus `sh_` prefix and `.lua`) becomes the `uniqueID`.

```lua
-- classes/sh_policechief.lua
CLASS.name = "Police Chief"
CLASS.description = "Leader of the police force"
CLASS.faction = FACTION_POLICE
CLASS.isDefault = false  -- Only one per faction should be true
CLASS.limit = 1  -- 0 = unlimited

-- Optional: Custom permission check
function CLASS:CanSwitchTo(client)
    local character = client:GetCharacter()

    -- Require specific flag
    if not character:HasFlags("C") then
        return false, "You need Chief rank"
    end

    -- Require playtime
    if client:GetUTime() < 86400 then
        return false, "You need 1 day of playtime"
    end

    return true
end

-- Optional: Called when player joins this class
function CLASS:OnSet(client)
    client:SetMaxHealth(150)
    client:SetHealth(150)
    client:Give("weapon_pistol")
end

-- Optional: Called when player leaves this class
function CLASS:OnLeave(client)
    client:StripWeapon("weapon_pistol")
    client:SetMaxHealth(100)
end
```

### Class Fields

```lua
CLASS.name = "Class Name"  -- Display name
CLASS.description = "Description"  -- Description shown in UI
CLASS.faction = FACTION_ID  -- REQUIRED: Which faction this class belongs to
CLASS.uniqueID = "classid"  -- Auto-set from filename
CLASS.index = 1  -- Auto-set numeric index
CLASS.isDefault = false  -- True for default class (one per faction)
CLASS.limit = 0  -- Player limit (0 = unlimited)
CLASS.plugin = "pluginid"  -- Auto-set if defined in plugin
```

## Complete Examples

### Police Faction Classes

```lua
-- classes/sh_officer.lua
CLASS.name = "Police Officer"
CLASS.description = "Standard police officer"
CLASS.faction = FACTION_POLICE
CLASS.isDefault = true  -- Default class for police
CLASS.limit = 0

function CLASS:OnSet(client)
    client:SetMaxHealth(100)
    client:Give("weapon_pistol")
end

-- classes/sh_sergeant.lua
CLASS.name = "Police Sergeant"
CLASS.description = "Experienced officer"
CLASS.faction = FACTION_POLICE
CLASS.limit = 3

function CLASS:CanSwitchTo(client)
    -- Require flag 'S' for sergeant
    return client:GetCharacter():HasFlags("S")
end

function CLASS:OnSet(client)
    client:SetMaxHealth(125)
    client:Give("weapon_pistol")
    client:Give("weapon_shotgun")
end

-- classes/sh_chief.lua
CLASS.name = "Police Chief"
CLASS.description = "Leader of the police force"
CLASS.faction = FACTION_POLICE
CLASS.limit = 1

function CLASS:CanSwitchTo(client)
    local character = client:GetCharacter()

    -- Require Chief flag
    if not character:HasFlags("C") then
        return false, "Insufficient rank"
    end

    -- Require minimum playtime (24 hours)
    if client:GetUTime() < 86400 then
        return false, "Not enough playtime"
    end

    return true
end

function CLASS:OnSet(client)
    client:SetMaxHealth(150)
    client:Give("weapon_pistol")
    client:Give("weapon_smg")
    client:ChatPrint("You are now the Police Chief!")
end

function CLASS:OnLeave(client)
    client:SetMaxHealth(100)
    client:ChatPrint("You are no longer the Police Chief")
end
```

### Class Switcher Command

```lua
ix.command.Add("BecomeClass", {
    description = "Join a class in your faction",
    arguments = ix.type.string,
    OnRun = function(self, client, className)
        local character = client:GetCharacter()
        local faction = character:GetFaction()

        -- Find matching class
        local targetClass
        for _, class in ipairs(ix.class.list) do
            if class.faction == faction and
               class.name:lower():find(className:lower()) then
                targetClass = class
                break
            end
        end

        if not targetClass then
            return "No class found matching '" .. className .. "' in your faction"
        end

        -- Check if can switch
        local can, reason = ix.class.CanSwitchTo(client, targetClass.index)
        if not can then
            return "Cannot join: " .. (reason or "unknown reason")
        end

        -- Join class
        if character:JoinClass(targetClass.index) then
            return "You joined: " .. targetClass.name
        else
            return "Failed to join class"
        end
    end
})
```

### Class Listing Menu

```lua
-- Show available classes for player's faction
function ShowClassMenu(client)
    local character = client:GetCharacter()
    local faction = character:GetFaction()

    net.Start("ClassMenu")
        local classes = {}

        for _, class in ipairs(ix.class.list) do
            if class.faction == faction then
                local players = ix.class.GetPlayers(class.index)
                local canJoin = ix.class.CanSwitchTo(client, class.index)

                table.insert(classes, {
                    name = class.name,
                    description = class.description,
                    limit = class.limit,
                    current = #players,
                    canJoin = canJoin,
                    index = class.index
                })
            end
        end

        net.WriteTable(classes)
    net.Send(client)
end
```

## Best Practices

### ✅ DO

- Always set one class per faction as `isDefault = true`
- Use `limit = 0` for unlimited classes
- Validate in `CanSwitchTo` for special requirements
- Check `ix.class.CanSwitchTo` before allowing joins
- Clean up in `OnLeave` what you set in `OnSet`
- Use meaningful class names and descriptions
- Store classes in `schema/classes/` or `plugins/yourplugin/classes/`

### ❌ DON'T

- Don't create classes without specifying a faction
- Don't forget to set a default class for each faction
- Don't bypass `CanSwitchTo` checks
- Don't call `JoinClass` without checking `CanSwitchTo`
- Don't create classes with generic names like "class1"
- Don't forget that `OnSet`/`OnLeave` only run on server
- Don't assume player is online in `OnLeave` - check IsValid

## Common Patterns

### Class with Equipment Requirements

```lua
function CLASS:CanSwitchTo(client)
    local character = client:GetCharacter()
    local inventory = character:GetInventory()

    -- Require specific item
    if not inventory:HasItem("police_badge") then
        return false, "You need a police badge"
    end

    return true
end
```

### Class with Attribute Requirements

```lua
function CLASS:CanSwitchTo(client)
    local character = client:GetCharacter()

    -- Require minimum strength
    if character:GetAttribute("str", 0) < 15 then
        return false, "Requires 15+ Strength"
    end

    return true
end
```

### Temporary Class Bonuses

```lua
function CLASS:OnSet(client)
    local character = client:GetCharacter()

    -- Add temporary attribute boost
    character:AddBoost(self.uniqueID .. "_str", "str", 5)
end

function CLASS:OnLeave(client)
    local character = client:GetCharacter()

    -- Remove temporary boost
    character:RemoveBoost(self.uniqueID .. "_str", "str")
end
```

## Common Issues

### "No default class set for faction"

**Cause**: Faction has no class with `isDefault = true`.

**Fix**: Mark one class as default:
```lua
-- At least one class per faction needs this
CLASS.isDefault = true
```

### Players Can't Join Any Class

**Cause**: `CanSwitchTo` returning false or class full.

**Fix**: Check implementation:
```lua
-- Debug CanSwitchTo
function CLASS:CanSwitchTo(client)
    print("Can join?", client:Name())
    -- Make sure to return true for at least default classes
    return true
end
```

### Class Attributes Not Removing

**Cause**: Forgetting to remove boosts in `OnLeave`.

**Fix**: Always clean up:
```lua
function CLASS:OnLeave(client)
    local character = client:GetCharacter()
    if character then
        character:RemoveBoost(self.uniqueID .. "_str", "str")
    end
end
```

## Related Hooks

### PlayerJoinedClass

Called when a player joins a class.

```lua
hook.Add("PlayerJoinedClass", "MyHook", function(client, class, oldClass)
    print(client:Name(), "joined class", class)
    if oldClass then
        print("Was previously in class", oldClass)
    end
end)
```

### CanPlayerJoinClass

Called to check if player can join a class.

```lua
hook.Add("CanPlayerJoinClass", "MyHook", function(client, class, classTable)
    -- Additional validation
    if client:Frags() < 10 then
        return false  -- Deny access
    end
    -- Don't return anything to allow other checks
end)
```

## See Also

- [Faction API](faction.md) - Faction system that classes belong to
- [Character API](character.md) - Character methods for classes
- [Flag API](flag.md) - Permission system for class restrictions
- Source: `gamemode/core/libs/sh_class.lua`
