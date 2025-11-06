# Creating Schema Classes

> **Reference**: `gamemode/core/libs/sh_class.lua`, `schema/classes/`

Classes are temporary jobs within factions. This guide shows how to create classes specifically for your schema.

## ⚠️ Important: Use Helix Class System

**Always use Helix's built-in class registration** rather than creating custom job systems. The framework provides:
- Automatic registration from `schema/classes/` folder
- Join/leave functionality
- Player limit management
- Permission checking
- Database persistence

## Core Concepts

### What are Schema Classes?

Classes are optional jobs within factions:
- **Police Faction** → Police Chief, Police Officer, Police Recruit
- **Medical Faction** → Doctor, Nurse, Paramedic
- **Resistance** → Squad Leader, Fighter, Scout

Key points:
- Classes are optional (characters can exist without a class)
- Characters can only join classes in their faction
- Joining a class automatically leaves the previous class
- Classes can have player limits

## Creating Classes

### File Location

**Place class files** in:
```
schema/classes/sh_classname.lua
```

**File naming convention**:
- `sh_` prefix (shared realm)
- Descriptive class name
- `.lua` extension

Examples:
- `sh_police_chief.lua`
- `sh_doctor.lua`
- `sh_squad_leader.lua`

### Basic Class Template

```lua
-- schema/classes/sh_officer.lua
CLASS.name = "Police Officer"
CLASS.description = "Standard patrol officer"
CLASS.faction = FACTION_POLICE  -- Faction index (required)
CLASS.limit = 8                 -- Max 8 officers at once

function CLASS:OnSet(client)
    client:SetHealth(100)
    client:Give("weapon_pistol")
end

function CLASS:OnRemoved(client)
    client:StripWeapon("weapon_pistol")
end
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't create custom job systems
JOBS = {}
JOBS["officer"] = {...}

-- WRONG: Don't bypass the class system
function CustomJobSystem()
    -- Don't do this!
end
```

## Required Properties

### Minimal Requirements

```lua
CLASS.name = "Class Name"          -- REQUIRED
CLASS.faction = FACTION_INDEX      -- REQUIRED (must be valid faction)
```

**⚠️ If missing faction**: Class will not load properly.

## Common Class Examples

### Police Chief

```lua
-- schema/classes/sh_police_chief.lua
CLASS.name = "Police Chief"
CLASS.description = "Leader of the police force"
CLASS.faction = FACTION_POLICE
CLASS.limit = 1  -- Only one chief
CLASS.pay = 200

CLASS.weapons = {
    "weapon_pistol",
    "weapon_shotgun"
}

function CLASS:CanSwitchTo(client)
    local character = client:GetCharacter()

    -- Must have chief flag
    if not character:HasFlags("c") then
        return false, "You are not authorized"
    end

    -- Must be rank 5+
    local rank = character:GetData("rank", 0)
    if rank < 5 then
        return false, "Requires rank 5"
    end

    return true
end

function CLASS:OnSet(client)
    -- Set stats
    client:SetHealth(150)
    client:SetArmor(100)
    client:SetRunSpeed(250)

    -- Give weapons
    for _, weapon in ipairs(self.weapons) do
        client:Give(weapon)
    end

    -- Give items
    local inventory = client:GetCharacter():GetInventory()
    inventory:Add("item_radio_command")

    -- Announce
    client:ChatPrint("You are now the Police Chief")
    PrintMessage(HUD_PRINTTALK, client:Name() .. " is now the Police Chief")
end

function CLASS:OnRemoved(client)
    for _, weapon in ipairs(self.weapons) do
        client:StripWeapon(weapon)
    end

    client:SetHealth(100)
    client:SetArmor(0)
    client:SetRunSpeed(240)
end
```

### Police Officer

```lua
-- schema/classes/sh_police_officer.lua
CLASS.name = "Police Officer"
CLASS.description = "Standard patrol officer"
CLASS.faction = FACTION_POLICE
CLASS.limit = 8
CLASS.pay = 100

function CLASS:OnSet(client)
    client:SetHealth(100)
    client:SetArmor(50)

    client:Give("weapon_pistol")
    client:Give("weapon_stunstick")

    local inventory = client:GetCharacter():GetInventory()
    inventory:Add("item_radio")
end

function CLASS:OnRemoved(client)
    client:StripWeapon("weapon_pistol")
    client:StripWeapon("weapon_stunstick")
end
```

### Medical Doctor

```lua
-- schema/classes/sh_doctor.lua
CLASS.name = "Doctor"
CLASS.description = "Medical professional"
CLASS.faction = FACTION_MEDICAL
CLASS.limit = 4
CLASS.pay = 150

function CLASS:OnSet(client)
    local inventory = client:GetCharacter():GetInventory()

    -- Give medical supplies
    inventory:Add("item_medkit")
    inventory:Add("item_medkit")
    inventory:Add("item_defibrillator")
    inventory:Add("item_bandage")

    -- Faster movement
    client:SetRunSpeed(260)

    client:ChatPrint("You are now a Doctor")
end

function CLASS:OnRemoved(client)
    client:SetRunSpeed(240)
end
```

### Resistance Squad Leader

```lua
-- schema/classes/sh_squad_leader.lua
CLASS.name = "Squad Leader"
CLASS.description = "Leads resistance squads"
CLASS.faction = FACTION_RESISTANCE
CLASS.limit = 2
CLASS.pay = 75

function CLASS:CanSwitchTo(client)
    local character = client:GetCharacter()

    -- Require leadership flag
    if not character:HasFlags("l") then
        return false, "You lack leadership training"
    end

    -- Require playtime
    local playtime = character:GetData("playtime", 0)
    if playtime < 3600 then  -- 1 hour
        return false, "Need 1 hour playtime"
    end

    return true
end

function CLASS:OnSet(client)
    local char = client:GetCharacter()
    local inventory = char:GetInventory()

    -- Leadership equipment
    inventory:Add("item_radio_encrypted")
    inventory:Add("item_map")
    inventory:Add("item_compass")

    client:SetHealth(125)
    client:SetRunSpeed(270)

    -- Announce to faction
    for _, ply in ipairs(player.GetAll()) do
        local plyChr = ply:GetCharacter()
        if plyChr and plyChr:GetFaction() == FACTION_RESISTANCE then
            ply:ChatPrint(client:Name() .. " is now a Squad Leader")
        end
    end
end

function CLASS:OnRemoved(client)
    client:SetHealth(100)
    client:SetRunSpeed(260)
end
```

## Class Properties

### Required Properties

```lua
CLASS.name = "Class Name"           -- Display name (REQUIRED)
CLASS.faction = FACTION_INDEX       -- Faction index (REQUIRED)
```

### Optional Properties

```lua
CLASS.description = "Description"   -- Shown in class selection
CLASS.limit = 0                     -- Max players (0 = unlimited)
CLASS.pay = 100                     -- Salary amount
CLASS.weapons = {...}               -- Weapons given on set
```

## Class Functions

### CanSwitchTo

**Reference**: `gamemode/core/libs/sh_class.lua:66`

Check if player can join this class:

```lua
function CLASS:CanSwitchTo(client)
    local character = client:GetCharacter()

    -- Check flag
    if not character:HasFlags("o") then
        return false, "You need officer flag"
    end

    -- Check rank
    local rank = character:GetData("rank", 0)
    if rank < self.requiredRank or 3 then
        return false, "Insufficient rank"
    end

    -- Check money
    if not character:HasMoney(self.joinCost or 0) then
        return false, "Costs " .. (self.joinCost or 0)
    end

    return true
end
```

### OnSet

Called when player joins the class:

```lua
function CLASS:OnSet(client)
    local character = client:GetCharacter()
    local inventory = character:GetInventory()

    -- Give weapons
    for _, weapon in ipairs(self.weapons or {}) do
        client:Give(weapon)
    end

    -- Give items
    inventory:Add("item_badge")
    inventory:Add("item_radio")

    -- Set stats
    client:SetHealth(self.health or 100)
    client:SetArmor(self.armor or 0)
    client:SetRunSpeed(self.runSpeed or 240)

    -- Charge join cost
    if self.joinCost and self.joinCost > 0 then
        character:TakeMoney(self.joinCost)
    end

    -- Notify
    client:ChatPrint("You are now a " .. self.name)
end
```

### OnRemoved

Called when player leaves the class:

```lua
function CLASS:OnRemoved(client)
    -- Remove weapons
    for _, weapon in ipairs(self.weapons or {}) do
        client:StripWeapon(weapon)
    end

    -- Reset stats
    client:SetHealth(100)
    client:SetArmor(0)
    client:SetRunSpeed(240)

    client:ChatPrint("You left " .. self.name)
end
```

## Advanced Class Features

### Paid Classes

```lua
CLASS.name = "Elite Officer"
CLASS.description = "Premium police role"
CLASS.faction = FACTION_POLICE
CLASS.limit = 2
CLASS.joinCost = 500  -- Costs $500 to join

function CLASS:CanSwitchTo(client)
    local character = client:GetCharacter()

    if not character:HasMoney(self.joinCost) then
        return false, "Costs " .. ix.currency.Get(self.joinCost)
    end

    return true
end

function CLASS:OnSet(client)
    local character = client:GetCharacter()

    -- Take money
    character:TakeMoney(self.joinCost)

    -- Give premium equipment
    client:SetHealth(150)
    client:SetArmor(100)
    client:Give("weapon_smg1")
end
```

### Timed Class Duration

```lua
CLASS.name = "Temporary Role"
CLASS.description = "30 minute assignment"
CLASS.faction = FACTION_CITIZEN
CLASS.limit = 1
CLASS.duration = 1800  -- 30 minutes

function CLASS:OnSet(client)
    -- Start timer
    timer.Create("ClassTimer_" .. client:SteamID(), self.duration, 1, function()
        if IsValid(client) then
            local character = client:GetCharacter()
            if character and character:GetClass() == self.index then
                character:KickClass()
                client:Notify("Your " .. self.name .. " shift has ended")
            end
        end
    end)

    client:Notify("You have " .. (self.duration / 60) .. " minutes")
end

function CLASS:OnRemoved(client)
    timer.Remove("ClassTimer_" .. client:SteamID())
end
```

### Rank-Based Requirements

```lua
CLASS.name = "Lieutenant"
CLASS.description = "Requires rank 4+"
CLASS.faction = FACTION_POLICE
CLASS.limit = 3
CLASS.requiredRank = 4

function CLASS:CanSwitchTo(client)
    local character = client:GetCharacter()
    local rank = character:GetData("rank", 0)

    if rank < self.requiredRank then
        return false, "Requires rank " .. self.requiredRank
    end

    return true
end
```

## Multiple Class Schema Example

### Police faction classes

```
schema/classes/
├── sh_police_chief.lua      -- Limit 1, requires flag
├── sh_police_lieutenant.lua -- Limit 2, requires rank
├── sh_police_officer.lua    -- Limit 8, basic
└── sh_police_recruit.lua    -- Unlimited, no requirements
```

### Police Recruit

```lua
-- sh_police_recruit.lua
CLASS.name = "Police Recruit"
CLASS.description = "Entry-level officer"
CLASS.faction = FACTION_POLICE
CLASS.limit = 0  -- Unlimited
CLASS.pay = 50

function CLASS:OnSet(client)
    client:Give("weapon_stunstick")
    client:SetHealth(75)

    client:ChatPrint("You are now a Recruit")
end

function CLASS:OnRemoved(client)
    client:StripWeapon("weapon_stunstick")
    client:SetHealth(100)
end
```

### Police Officer

```lua
-- sh_police_officer.lua
CLASS.name = "Police Officer"
CLASS.description = "Standard patrol officer"
CLASS.faction = FACTION_POLICE
CLASS.limit = 8
CLASS.pay = 100

function CLASS:CanSwitchTo(client)
    local rank = client:GetCharacter():GetData("rank", 0)
    return rank >= 2, "Requires rank 2"
end

function CLASS:OnSet(client)
    client:Give("weapon_stunstick")
    client:Give("weapon_pistol")
    client:SetHealth(100)
    client:SetArmor(50)
end
```

### Police Lieutenant

```lua
-- sh_police_lieutenant.lua
CLASS.name = "Lieutenant"
CLASS.description = "Senior officer"
CLASS.faction = FACTION_POLICE
CLASS.limit = 2
CLASS.pay = 150

function CLASS:CanSwitchTo(client)
    local char = client:GetCharacter()
    local rank = char:GetData("rank", 0)

    if rank < 4 then
        return false, "Requires rank 4"
    end

    if not char:HasFlags("l") then
        return false, "Requires lieutenant flag"
    end

    return true
end

function CLASS:OnSet(client)
    client:SetHealth(125)
    client:SetArmor(75)
    client:Give("weapon_pistol")
    client:Give("weapon_smg1")
end
```

## Best Practices

### ✅ DO

- Place class files in `schema/classes/` folder
- Use `sh_` prefix for class files
- Always set CLASS.faction to valid faction index
- Use CLASS:CanSwitchTo() for requirements
- Use CLASS:OnSet() for equipment/stats
- Use CLASS:OnRemoved() for cleanup
- Set limits for leadership/special roles
- Use descriptive class names

### ❌ DON'T

- Don't create classes without valid faction
- Don't forget to check faction match
- Don't implement custom job systems
- Don't forget cleanup in OnRemoved()
- Don't give unlimited powerful classes
- Don't bypass class system with custom code

## Common Patterns

### Class Selection Command

```lua
-- In schema or plugin
ix.command.Add("BecomeClass", {
    description = "Join a class",
    arguments = {ix.type.string},
    OnRun = function(self, client, className)
        local character = client:GetCharacter()
        local factionID = character:GetFaction()

        -- Find matching class
        local targetClass
        for _, class in ipairs(ix.class.list) do
            if string.lower(class.name) == string.lower(className) then
                if class.faction == factionID then
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

### Class-Specific Abilities

```lua
-- In schema hooks
function Schema:KeyPress(client, key)
    local character = client:GetCharacter()
    if not character then return end

    local classIndex = character:GetClass()
    if not classIndex then return end

    local class = ix.class.Get(classIndex)

    -- Squad Leader ability
    if class.uniqueID == "squad_leader" and key == IN_RELOAD then
        -- Rally nearby allies
        for _, ply in ipairs(ents.FindInSphere(client:GetPos(), 500)) do
            if ply:IsPlayer() and ply:GetCharacter() then
                if ply:GetCharacter():GetFaction() == character:GetFaction() then
                    ply:SetHealth(math.min(ply:Health() + 25, ply:GetMaxHealth()))
                end
            end
        end

        client:ChatPrint("Rallied nearby allies!")
    end
end
```

### Automatic Class Assignment

```lua
-- In faction file
function FACTION:OnCharacterCreated(client, character)
    -- Auto-assign to recruit class
    timer.Simple(1, function()
        if IsValid(client) and character then
            local recruitClass = ix.class.Get("police_recruit")
            if recruitClass then
                character:JoinClass(recruitClass.index)
            end
        end
    end)
end
```

## Common Issues

### Class not appearing

**Cause**: File not in `schema/classes/` or wrong naming
**Fix**: Place in `schema/classes/sh_name.lua`

### Cannot join class

**Cause**: Wrong faction or limit reached
**Fix**: Verify CLASS.faction matches character's faction

### Class index is nil

**Cause**: Accessing CLASS.index too early
**Fix**: Access after schema loads or use class getter

### OnRemoved not called

**Cause**: Player disconnected or class code error
**Fix**: Always handle edge cases in OnRemoved

## See Also

- [Class System](../systems/classes.md) - Detailed class system reference
- [Faction System](../systems/factions.md) - Faction management
- [Schema Factions](factions.md) - Creating factions
- [Command System](../systems/commands.md) - Class commands
- Source: `gamemode/core/libs/sh_class.lua`
