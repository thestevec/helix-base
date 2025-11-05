# Class System

> **Reference**: `gamemode/core/libs/sh_class.lua`, `gamemode/core/meta/sh_character.lua`

The Helix class system provides temporary job assignments within factions (like "Police Chief" within "Police Force").

## ⚠️ Important: Use Built-in Helix Functions

**Always use Helix's built-in class system** rather than creating custom job systems. The framework provides:
- Automatic class registration
- Join/leave functionality
- Player limits per class
- Permission checking via CanSwitchTo
- Database persistence

## Core Concepts

### Class vs Faction

- **Faction**: Permanent team (e.g., "Police") - set at character creation
- **Class**: Temporary job within faction (e.g., "Police Chief") - can be changed in-game

```
Character: John Doe
├── Faction: Police (permanent)
└── Class: Police Chief (can switch to Police Officer)
```

### Class Assignment

- Classes are optional (characters can have no class)
- Characters can only join classes in their faction
- Classes can have player limits
- Joining a class automatically leaves previous class

## Creating Classes

### File Location

**Reference**: `gamemode/core/libs/sh_class.lua:24`

**Framework automatically loads classes** from:
- `schema/classes/` - Schema classes
- `plugins/yourplugin/classes/` - Plugin classes

### Basic Class

```lua
-- File: schema/classes/sh_officer.lua
CLASS.name = "Police Officer"
CLASS.description = "Standard patrol officer"
CLASS.faction = FACTION_POLICE  -- Required: faction index
CLASS.limit = 4  -- Optional: max 4 players (0 = unlimited)

-- Optional: Called when player joins class
function CLASS:OnSet(client)
    client:SetHealth(100)
    client:Give("weapon_pistol")
end

-- Optional: Called when player leaves class
function CLASS:OnRemoved(client)
    client:StripWeapon("weapon_pistol")
end

-- Optional: Check if player can join
function CLASS:CanSwitchTo(client)
    local character = client:GetCharacter()

    -- Example: Require rank
    if character:GetData("rank", 0) < 3 then
        return false, "You need rank 3 or higher"
    end

    return true
end
```

**File naming**: `sh_classname.lua` where `classname` is descriptive name.

## Class Properties

### Required Properties

```lua
CLASS.name = "Class Name"        -- Display name (REQUIRED)
CLASS.faction = FACTION_INDEX    -- Faction index (REQUIRED)
```

**⚠️ Class without valid faction will not load**:
```lua
ErrorNoHalt("Class 'classname' does not have a valid faction!")
```

### Optional Properties

```lua
CLASS.description = "Description" -- Description text
CLASS.limit = 0                   -- Max players (0 = unlimited)
CLASS.pay = 100                   -- Salary amount
CLASS.weapons = {...}             -- Weapons given on spawn
```

### Automatic Properties

```lua
CLASS.index = 1                   -- Numeric index
CLASS.uniqueID = "officer"        -- From filename
CLASS.plugin = "pluginname"       -- If from plugin
```

## Joining Classes

### Use Character Methods

**Reference**: `gamemode/core/libs/sh_class.lua:148`

```lua
-- SERVER ONLY
character:JoinClass(classIndex)

-- Example
local classIndex = 1  -- Officer class
if character:JoinClass(classIndex) then
    client:Notify("Joined class!")
else
    client:Notify("Cannot join class")
end

-- Leave class
character:JoinClass(nil)  -- or character:KickClass()
```

### Checking if Can Join

**Reference**: `gamemode/core/libs/sh_class.lua:82`

```lua
local canJoin, reason = ix.class.CanSwitchTo(client, classIndex)

if canJoin then
    character:JoinClass(classIndex)
else
    client:Notify("Cannot join: " .. (reason or "Unknown"))
end

-- Framework automatically checks:
-- - Player is in correct faction
-- - Class limit not reached
-- - Player not already in this class
-- - CanPlayerJoinClass hook
-- - CLASS:CanSwitchTo()
```

## Class Functions

### CanSwitchTo

**Reference**: `gamemode/core/libs/sh_class.lua:66`

Default allows anyone in faction to join:

```lua
-- Default implementation
function CLASS:CanSwitchTo(client)
    return true
end

-- Custom restrictions
function CLASS:CanSwitchTo(client)
    local character = client:GetCharacter()

    -- Require specific flag
    if not character:HasFlags("o") then
        return false, "You need officer flag"
    end

    -- Require rank
    local rank = character:GetData("rank", 0)
    if rank < self.requiredRank then
        return false, "Insufficient rank"
    end

    -- Require time played
    local playtime = character:GetData("playtime", 0)
    if playtime < 3600 then  -- 1 hour
        return false, "Need 1 hour playtime"
    end

    return true
end
```

### OnSet

Called when player joins class:

```lua
function CLASS:OnSet(client)
    -- Give weapons
    for _, weapon in ipairs(self.weapons or {}) do
        client:Give(weapon)
    end

    -- Set health
    client:SetHealth(self.health or 100)

    -- Notify
    client:ChatPrint("You are now a " .. self.name)
end
```

### OnRemoved

Called when player leaves class:

```lua
function CLASS:OnRemoved(client)
    -- Remove weapons
    for _, weapon in ipairs(self.weapons or {}) do
        client:StripWeapon(weapon)
    end

    -- Reset health
    client:SetHealth(100)

    client:ChatPrint("You left " .. self.name)
end
```

## Getting Classes

### Use Built-in Functions

```lua
-- Get class by index
local class = ix.class.Get(classIndex)
local class = ix.class.list[classIndex]

-- Get character's current class
local classIndex = character:GetClass()
if classIndex then
    local class = ix.class.Get(classIndex)
    print("Class:", class.name)
else
    print("No class assigned")
end

-- Get all players in a class
local players = ix.class.GetPlayers(classIndex)
for _, ply in ipairs(players) do
    print(ply:Name())
end
```

## Complete Examples

### Police Chief Class

```lua
-- File: schema/classes/sh_chief.lua
CLASS.name = "Police Chief"
CLASS.description = "Leader of the police force"
CLASS.faction = FACTION_POLICE
CLASS.limit = 1  -- Only one chief
CLASS.pay = 200

CLASS.weapons = {
    "weapon_pistol",
    "weapon_shotgun"
}

CLASS.health = 150
CLASS.armor = 100

function CLASS:CanSwitchTo(client)
    local character = client:GetCharacter()

    -- Must have chief flag
    if not character:HasFlags("c") then
        return false, "You are not authorized for this position"
    end

    -- Must be rank 5+
    local rank = character:GetData("rank", 0)
    if rank < 5 then
        return false, "You need rank 5 or higher"
    end

    return true
end

function CLASS:OnSet(client)
    -- Set appearance
    client:SetHealth(self.health)
    client:SetArmor(self.armor)

    -- Give equipment
    for _, weapon in ipairs(self.weapons) do
        client:Give(weapon)
    end

    -- Give items
    local inventory = client:GetCharacter():GetInventory()
    inventory:Add("item_radio_command")

    -- Notify
    client:ChatPrint("You are now the Police Chief!")
    PrintMessage(HUD_PRINTTALK, client:Name() .. " is now the Police Chief")
end

function CLASS:OnRemoved(client)
    -- Remove equipment
    for _, weapon in ipairs(self.weapons) do
        client:StripWeapon(weapon)
    end

    client:ChatPrint("You are no longer the Police Chief")
end
```

### Medical Doctor Class

```lua
-- File: schema/classes/sh_doctor.lua
CLASS.name = "Doctor"
CLASS.description = "Medical professional"
CLASS.faction = FACTION_MEDICAL
CLASS.limit = 3
CLASS.pay = 150

function CLASS:CanSwitchTo(client)
    -- Anyone in medical faction can be doctor
    return true
end

function CLASS:OnSet(client)
    local inventory = client:GetCharacter():GetInventory()

    -- Give medical supplies
    inventory:Add("item_medkit")
    inventory:Add("item_medkit")
    inventory:Add("item_defibrillator")

    client:SetRunSpeed(260)  -- Faster than normal
end

function CLASS:OnRemoved(client)
    client:SetRunSpeed(240)  -- Reset speed
end
```

## Class Commands

### Example Switch Command

```lua
ix.command.Add("BecomeClass", {
    description = "Join a class in your faction",
    arguments = {ix.type.string},  -- Class name
    OnRun = function(self, client, className)
        local character = client:GetCharacter()

        -- Find class by name
        local targetClass
        for _, class in ipairs(ix.class.list) do
            if string.lower(class.name) == string.lower(className) then
                if class.faction == character:GetFaction() then
                    targetClass = class
                    break
                end
            end
        end

        if not targetClass then
            return "Class not found in your faction"
        end

        -- Check if can join
        local canJoin, reason = ix.class.CanSwitchTo(client, targetClass.index)
        if not canJoin then
            return reason or "Cannot join class"
        end

        -- Join class
        if character:JoinClass(targetClass.index) then
            return "You are now a " .. targetClass.name
        else
            return "Failed to join class"
        end
    end
})
```

## Class Hooks

### CanPlayerJoinClass

```lua
function PLUGIN:CanPlayerJoinClass(client, class, classInfo)
    local character = client:GetCharacter()

    -- Prevent joining if character is injured
    if character:GetData("injured") then
        client:Notify("You cannot switch classes while injured")
        return false
    end

    -- Require payment for certain classes
    if classInfo.joinCost and classInfo.joinCost > 0 then
        if not character:HasMoney(classInfo.joinCost) then
            client:Notify("You need $" .. classInfo.joinCost)
            return false
        end

        character:TakeMoney(classInfo.joinCost)
    end
end
```

### PlayerJoinedClass

```lua
function PLUGIN:PlayerJoinedClass(client, class, oldClass)
    local character = client:GetCharacter()

    -- Log class change
    ix.log.Add(client, "joinedClass", class)

    -- Announce
    local classInfo = ix.class.Get(class)
    if classInfo then
        for _, ply in ipairs(player.GetAll()) do
            ply:ChatPrint(client:Name() .. " became a " .. classInfo.name)
        end
    end
end
```

## Best Practices

### ✅ DO

- Store classes in `schema/classes/` or `plugins/yourplugin/classes/`
- Always set CLASS.faction to valid faction index
- Use CLASS:CanSwitchTo() for join requirements
- Use CLASS:OnSet() for equipment/effects
- Use CLASS:OnRemoved() to clean up
- Set class limits for leadership roles
- Use `character:JoinClass()` to assign classes

### ❌ DON'T

- Don't create classes without valid faction
- Don't forget to check faction match before joining
- Don't implement custom class systems
- Don't modify character's class directly
- Don't forget to clean up in OnRemoved()

## Common Patterns

### Paid Classes

```lua
CLASS.joinCost = 500

function CLASS:CanSwitchTo(client)
    local character = client:GetCharacter()

    if not character:HasMoney(self.joinCost) then
        return false, "Costs $" .. self.joinCost
    end

    return true
end

function CLASS:OnSet(client)
    client:GetCharacter():TakeMoney(self.joinCost)
end
```

### Timed Class Duration

```lua
function CLASS:OnSet(client)
    -- Auto-remove after 30 minutes
    timer.Create("ClassTimer_" .. client:SteamID(), 1800, 1, function()
        if IsValid(client) then
            local character = client:GetCharacter()
            if character and character:GetClass() == self.index then
                character:KickClass()
                client:Notify("Your " .. self.name .. " shift has ended")
            end
        end
    end)
end

function CLASS:OnRemoved(client)
    timer.Remove("ClassTimer_" .. client:SteamID())
end
```

## See Also

- [Faction System](factions.md) - Team management
- [Character System](character.md) - Character class assignment
- Class System: `gamemode/core/libs/sh_class.lua`
