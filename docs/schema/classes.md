# Creating Classes for Your Schema

> **Reference**: `gamemode/core/libs/sh_class.lua`, `schema/classes/`

This guide shows you how to create custom classes (jobs/roles) for your schema. Classes are temporary positions within factions that characters can join and leave.

## ⚠️ Important: Use Helix Class System

**Always create classes in `schema/classes/`** rather than implementing custom job systems. The framework provides:
- Automatic class registration from files
- Join/leave functionality with validation
- Player limits per class
- Permission checking integration
- Database persistence of class assignment

## Core Concepts

### Class vs Faction

Understanding the difference:

```
Faction (Permanent)          Class (Temporary)
─────────────────           ─────────────────
Police Force         ───>   Police Chief
                     ───>   Police Officer
                     ───>   Police Cadet

Medical Staff        ───>   Head Doctor
                     ───>   Surgeon
                     ───>   Nurse
```

- **Faction**: Set at character creation, permanent
- **Class**: Can be changed in-game, temporary job/role

### When to Use Classes

Use classes for:
- Leadership positions (Chief, Captain, Leader)
- Specialized roles (Sniper, Engineer, Medic)
- Ranks within factions (Recruit, Veteran, Elite)
- Limited positions (1 Mayor, 2 Judges, 4 SWAT)

**Don't use classes** if you just need different loadouts - use character data or items instead.

## Creating Your First Class

### Step 1: Create Class File

Create a new file in `schema/classes/`:

```lua
-- File: schema/classes/sh_officer.lua
CLASS.name = "Police Officer"
CLASS.description = "Standard patrol officer"
CLASS.faction = FACTION_POLICE  -- Must match a faction index
CLASS.limit = 8  -- Max 8 officers online (0 = unlimited)

-- Optional: Equipment given on spawn
CLASS.weapons = {
    "weapon_pistol"
}

-- Called when player joins this class
function CLASS:OnSet(client)
    client:SetHealth(100)
    client:SetArmor(50)

    -- Give equipment
    for _, weapon in ipairs(self.weapons) do
        client:Give(weapon)
    end

    client:ChatPrint("You are now a " .. self.name)
end

-- Called when player leaves this class
function CLASS:OnRemoved(client)
    -- Remove class weapons
    for _, weapon in ipairs(self.weapons) do
        client:StripWeapon(weapon)
    end

    client:SetArmor(0)
    client:ChatPrint("You are no longer a " .. self.name)
end

-- Store class index for later use
CLASS_OFFICER = CLASS.index
```

### Step 2: Create Join Command

Classes need commands or UI for players to join them:

```lua
-- File: schema/commands/sh_becomeclass.lua
ix.command.Add("BecomeClass", {
    description = "Join a class in your faction",
    arguments = {ix.type.string},  -- Class name
    OnRun = function(self, client, className)
        local character = client:GetCharacter()
        if not character then
            return "No active character"
        end

        -- Find class by name in player's faction
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

### Step 3: Test Your Class

1. Restart server
2. Create character in the faction
3. Use `/becomeclass officer` command
4. Verify class equipment and effects

## Class Properties

### Required Properties

```lua
CLASS.name = "Class Name"        -- REQUIRED: Display name
CLASS.faction = FACTION_INDEX    -- REQUIRED: Faction index
```

**⚠️ Missing faction will cause class to not load!**

### Common Properties

```lua
CLASS.description = "Description"  -- Shown in class selection
CLASS.limit = 0                    -- Max players (0 = unlimited)
CLASS.pay = 100                    -- Additional salary
CLASS.weapons = {}                 -- Weapons given on spawn
CLASS.health = 100                 -- Health when joining
CLASS.armor = 0                    -- Armor when joining
```

## Complete Class Examples

### Police Chief (Leadership)

```lua
-- File: schema/classes/sh_chief.lua
CLASS.name = "Police Chief"
CLASS.description = "Leader of the police force"
CLASS.faction = FACTION_POLICE
CLASS.limit = 1  -- Only one chief allowed

CLASS.weapons = {
    "weapon_pistol",
    "weapon_shotgun",
    "weapon_stunstick"
}

CLASS.health = 150
CLASS.armor = 100
CLASS.pay = 200  -- Higher salary

-- Custom join requirements
function CLASS:CanSwitchTo(client)
    local character = client:GetCharacter()

    -- Must have chief flag
    if not character:HasFlags("c") then
        return false, "You need the Chief flag to join this class"
    end

    -- Must have minimum rank
    local rank = character:GetData("rank", 0)
    if rank < 5 then
        return false, "You need rank 5 or higher"
    end

    -- Must have playtime
    local playtime = character:GetData("playtime", 0)
    if playtime < 3600 then  -- 1 hour
        return false, "You need at least 1 hour of playtime"
    end

    return true
end

function CLASS:OnSet(client)
    -- Set stats
    client:SetHealth(self.health)
    client:SetArmor(self.armor)

    -- Give all weapons
    for _, weapon in ipairs(self.weapons) do
        client:Give(weapon)
    end

    -- Give special equipment
    local inventory = client:GetCharacter():GetInventory()
    inventory:Add("item_radio_command")
    inventory:Add("item_keycard_chief")

    -- Announce
    for _, ply in ipairs(player.GetAll()) do
        ply:ChatPrint(client:Name() .. " is now the Police Chief!")
    end
end

function CLASS:OnRemoved(client)
    -- Remove weapons
    for _, weapon in ipairs(self.weapons) do
        client:StripWeapon(weapon)
    end

    -- Reset stats
    client:SetHealth(100)
    client:SetArmor(0)

    -- Announce
    for _, ply in ipairs(player.GetAll()) do
        ply:ChatPrint(client:Name() .. " is no longer the Police Chief")
    end
end

CLASS_CHIEF = CLASS.index
```

### SWAT Operative (Limited Special Role)

```lua
-- File: schema/classes/sh_swat.lua
CLASS.name = "SWAT Operative"
CLASS.description = "Special weapons and tactics officer"
CLASS.faction = FACTION_POLICE
CLASS.limit = 4  -- Only 4 SWAT members

CLASS.weapons = {
    "weapon_ar2",
    "weapon_pistol",
    "weapon_stunstick"
}

CLASS.health = 150
CLASS.armor = 100
CLASS.pay = 150

function CLASS:CanSwitchTo(client)
    local character = client:GetCharacter()

    -- Must have SWAT flag
    if not character:HasFlags("w") then
        return false, "You need SWAT certification"
    end

    -- Must be rank 3+
    local rank = character:GetData("rank", 0)
    if rank < 3 then
        return false, "You need rank 3 to join SWAT"
    end

    return true
end

function CLASS:OnSet(client)
    client:SetHealth(self.health)
    client:SetArmor(self.armor)
    client:SetRunSpeed(260)  -- Faster movement

    for _, weapon in ipairs(self.weapons) do
        client:Give(weapon)
    end

    -- Give equipment
    local inventory = client:GetCharacter():GetInventory()
    inventory:Add("item_flashbang")
    inventory:Add("item_flashbang")

    client:ChatPrint("You are now SWAT. Coordinate with your team!")
end

function CLASS:OnRemoved(client)
    for _, weapon in ipairs(self.weapons) do
        client:StripWeapon(weapon)
    end

    client:SetHealth(100)
    client:SetArmor(0)
    client:SetRunSpeed(240)  -- Reset speed
end

CLASS_SWAT = CLASS.index
```

### Medic (Support Role)

```lua
-- File: schema/classes/sh_medic.lua
CLASS.name = "Field Medic"
CLASS.description = "Medical support specialist"
CLASS.faction = FACTION_RESISTANCE
CLASS.limit = 3

CLASS.weapons = {}  -- No weapons, medics focus on healing

function CLASS:CanSwitchTo(client)
    -- Anyone in resistance can be medic
    return true
end

function CLASS:OnSet(client)
    local inventory = client:GetCharacter():GetInventory()

    -- Give medical supplies
    inventory:Add("item_medkit")
    inventory:Add("item_medkit")
    inventory:Add("item_bandage")
    inventory:Add("item_bandage")
    inventory:Add("item_defibrillator")

    -- Faster movement
    client:SetRunSpeed(260)

    client:ChatPrint("You are now a Field Medic. Help your team!")
end

function CLASS:OnRemoved(client)
    client:SetRunSpeed(240)  -- Reset speed
end

-- Passive health regeneration
function CLASS:OnThink(client)
    if client:Health() < client:GetMaxHealth() then
        client:SetHealth(math.min(client:Health() + 0.05, client:GetMaxHealth()))
    end
end

CLASS_MEDIC = CLASS.index
```

### Engineer (Resource Role)

```lua
-- File: schema/classes/sh_engineer.lua
CLASS.name = "Engineer"
CLASS.description = "Technical specialist and builder"
CLASS.faction = FACTION_RESISTANCE
CLASS.limit = 2

CLASS.weapons = {
    "weapon_physcannon"
}

function CLASS:OnSet(client)
    local inventory = client:GetCharacter():GetInventory()

    -- Give engineering tools
    inventory:Add("item_toolbox")
    inventory:Add("item_welder")
    inventory:Add("item_metal_scrap")
    inventory:Add("item_metal_scrap")

    for _, weapon in ipairs(self.weapons) do
        client:Give(weapon)
    end

    -- Set data for engineer abilities
    client:GetCharacter():SetData("canBuild", true)
end

function CLASS:OnRemoved(client)
    for _, weapon in ipairs(self.weapons) do
        client:StripWeapon(weapon)
    end

    client:GetCharacter():SetData("canBuild", false)
end

CLASS_ENGINEER = CLASS.index
```

## Class Functions

### CanSwitchTo

**Reference**: `gamemode/core/libs/sh_class.lua:66`

Check if player can join this class:

```lua
function CLASS:CanSwitchTo(client)
    local character = client:GetCharacter()

    -- Check flag
    if not character:HasFlags("s") then
        return false, "You need the sniper flag"
    end

    -- Check rank
    local rank = character:GetData("rank", 0)
    if rank < self.requiredRank or 3 then
        return false, "Insufficient rank: " .. self.requiredRank
    end

    -- Check money (paid classes)
    if self.joinCost and self.joinCost > 0 then
        if not character:HasMoney(self.joinCost) then
            return false, "Costs $" .. self.joinCost
        end
    end

    -- Check time requirement
    local playtime = character:GetData("playtime", 0)
    if playtime < 1800 then  -- 30 minutes
        return false, "Need 30 minutes playtime"
    end

    return true
end
```

### OnSet

**Reference**: `gamemode/core/libs/sh_class.lua` (called when joining)

Called when player successfully joins this class:

```lua
function CLASS:OnSet(client)
    -- Set health/armor
    client:SetHealth(self.health or 100)
    client:SetArmor(self.armor or 0)

    -- Give weapons
    for _, weapon in ipairs(self.weapons or {}) do
        client:Give(weapon)
    end

    -- Give items
    local inventory = client:GetCharacter():GetInventory()
    for _, itemID in ipairs(self.items or {}) do
        inventory:Add(itemID)
    end

    -- Set movement
    if self.runSpeed then
        client:SetRunSpeed(self.runSpeed)
    end

    -- Take payment if applicable
    if self.joinCost and self.joinCost > 0 then
        client:GetCharacter():TakeMoney(self.joinCost)
    end

    -- Set class data
    client:GetCharacter():SetData("joinedClass", CurTime())

    -- Notify
    client:ChatPrint("You are now a " .. self.name)
end
```

### OnRemoved

Called when player leaves this class:

```lua
function CLASS:OnRemoved(client)
    -- Remove weapons
    for _, weapon in ipairs(self.weapons or {}) do
        client:StripWeapon(weapon)
    end

    -- Reset stats
    client:SetHealth(100)
    client:SetArmor(0)

    -- Reset speeds
    client:SetRunSpeed(240)
    client:SetWalkSpeed(100)

    -- Clear class data
    client:GetCharacter():SetData("joinedClass", nil)

    -- Notify
    client:ChatPrint("You left " .. self.name)
end
```

### OnThink

Called every tick for players in this class:

```lua
function CLASS:OnThink(client)
    local character = client:GetCharacter()
    if not character then return end

    -- Example: Passive effects
    if self.uniqueID == "medic" then
        -- Health regeneration
        if client:Health() < client:GetMaxHealth() then
            client:SetHealth(math.min(client:Health() + 0.05, client:GetMaxHealth()))
        end
    end

    -- Example: Resource gathering
    if self.uniqueID == "miner" then
        local miningPower = character:GetData("miningPower", 0)
        if miningPower > 0 then
            character:SetData("miningPower", miningPower - 0.01)
        end
    end
end
```

## Advanced Class Features

### Paid Classes

```lua
CLASS.joinCost = 500

function CLASS:CanSwitchTo(client)
    local character = client:GetCharacter()

    if not character:HasMoney(self.joinCost) then
        return false, "This class costs $" .. self.joinCost
    end

    return true
end

function CLASS:OnSet(client)
    -- Take money when joining
    client:GetCharacter():TakeMoney(self.joinCost)
end
```

### Time-Limited Classes

```lua
function CLASS:OnSet(client)
    -- Auto-remove after 30 minutes
    timer.Create("ClassDuration_" .. client:SteamID(), 1800, 1, function()
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
    timer.Remove("ClassDuration_" .. client:SteamID())
end
```

### Rank-Based Classes

```lua
-- File: schema/classes/sh_lieutenant.lua
CLASS.name = "Lieutenant"
CLASS.faction = FACTION_POLICE
CLASS.requiredRank = 4

function CLASS:CanSwitchTo(client)
    local rank = client:GetCharacter():GetData("rank", 0)
    if rank < self.requiredRank then
        return false, "You need rank " .. self.requiredRank
    end
    return true
end
```

## Best Practices

### ✅ DO

- Create one class file per class in `schema/classes/`
- Use `sh_` prefix for automatic loading
- Always set CLASS.name and CLASS.faction
- Store CLASS.index in global (e.g., `CLASS_CHIEF`)
- Use limits for leadership/special roles
- Implement CanSwitchTo for requirements
- Clean up in OnRemoved
- Provide clear descriptions
- Balance class equipment
- Test class limits with multiple players

### ❌ DON'T

- Don't create classes without valid faction
- Don't forget to set player limits on special classes
- Don't implement custom class systems
- Don't modify character faction from class
- Don't forget to remove weapons in OnRemoved
- Don't allow unlimited special classes
- Don't bypass CanSwitchTo checks
- Don't forget cleanup timers in OnRemoved

## Common Patterns

### Auto-Promotion System

```lua
-- File: schema/sv_schema.lua
hook.Add("PlayerDeath", "ClassAutoKick", function(victim)
    local character = victim:GetCharacter()
    if character and character:GetClass() then
        -- Remove from class on death
        character:KickClass()
    end
end)
```

### Class-Specific Commands

```lua
ix.command.Add("ClassAbility", {
    description = "Use your class ability",
    OnRun = function(self, client)
        local character = client:GetCharacter()
        local classIndex = character:GetClass()

        if not classIndex then
            return "You don't have a class"
        end

        local class = ix.class.Get(classIndex)

        if class.uniqueID == "medic" then
            -- Medic heal ability
            return "Healing nearby players..."
        elseif class.uniqueID == "engineer" then
            -- Engineer build ability
            return "Opening build menu..."
        end
    end
})
```

### Class Whiteboard

```lua
-- Show all players in each class
ix.command.Add("ClassList", {
    description = "Show all active classes",
    OnRun = function(self, client)
        for classIndex, class in ipairs(ix.class.list) do
            local players = ix.class.GetPlayers(classIndex)
            if #players > 0 then
                client:ChatPrint(class.name .. " (" .. #players .. "/" .. (class.limit == 0 and "∞" or class.limit) .. "):")
                for _, ply in ipairs(players) do
                    client:ChatPrint("  - " .. ply:Name())
                end
            end
        end
    end
})
```

## See Also

- [Class System](../systems/classes.md) - Core class system reference
- [Factions](factions.md) - Creating factions for your schema
- [Commands](commands.md) - Creating class join commands
- [Schema Structure](structure.md) - Schema directory layout
- Source: `gamemode/core/libs/sh_class.lua`
