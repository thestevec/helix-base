# Complete Class Examples

> **Reference**: `gamemode/core/libs/sh_class.lua`, `gamemode/core/meta/sh_character.lua`

This document provides complete, working examples of Helix classes from basic to advanced.

## ⚠️ Important: Use Built-in Class System

**Always use Helix's class system** rather than creating custom job management. The framework provides:
- Automatic class registration
- Player limit enforcement
- Join/leave functionality
- Permission checking
- Database persistence
- Integration with factions

## Example 1: Basic Class

The simplest class definition.

### File Location

**Place in**: `schema/classes/sh_officer.lua`

The filename determines the class unique ID (`sh_officer.lua` → `"officer"`).

### Complete Code

```lua
CLASS.name = "Police Officer"
CLASS.description = "Standard patrol officer"
CLASS.faction = FACTION_POLICE  -- Must match faction index
CLASS.limit = 4  -- Maximum 4 players can be this class

-- Called when player joins this class
function CLASS:OnSet(client)
    -- Give equipment
    client:Give("weapon_pistol")
    client:Give("weapon_stunstick")

    -- Set appearance
    client:SetHealth(100)
    client:SetArmor(50)

    -- Notify player
    client:Notify("You are now a Police Officer")
end

-- Called when player leaves this class
function CLASS:OnRemoved(client)
    -- Remove equipment
    client:StripWeapon("weapon_pistol")
    client:StripWeapon("weapon_stunstick")

    client:Notify("You left the Police Officer class")
end

-- Store class index for use in code
CLASS_OFFICER = CLASS.index
```

### Key Points

- **Filename**: `sh_officer.lua` creates class ID `"officer"`
- **Required**: `name`, `faction`
- **`limit`**: Maximum players (0 = unlimited)
- **`OnSet()`**: Called when joining class
- **`OnRemoved()`**: Called when leaving class

## Example 2: Class with Requirements

A class that requires specific conditions to join.

### File Location

**Place in**: `schema/classes/sh_chief.lua`

### Complete Code

```lua
CLASS.name = "Police Chief"
CLASS.description = "Leader of the police force"
CLASS.faction = FACTION_POLICE
CLASS.limit = 1  -- Only one chief allowed

-- Required rank
CLASS.requiredRank = 5

-- Custom weapons
CLASS.weapons = {
    "weapon_pistol",
    "weapon_smg1",
    "weapon_stunstick"
}

-- Check if player can join
function CLASS:CanSwitchTo(client)
    local character = client:GetCharacter()

    if !character then
        return false, "No character"
    end

    -- Check rank
    local rank = character:GetData("rank", 0)
    if rank < self.requiredRank then
        return false, "You need rank " .. self.requiredRank .. " or higher"
    end

    -- Check flag
    if !character:HasFlags("c") then  -- 'c' for chief
        return false, "You need chief authorization"
    end

    -- Check playtime
    local playtime = character:GetData("playtime", 0)
    if playtime < 7200 then  -- 2 hours
        return false, "You need at least 2 hours of playtime"
    end

    return true
end

-- Called when joining
function CLASS:OnSet(client)
    local character = client:GetCharacter()

    -- Give all weapons
    for _, weapon in ipairs(self.weapons) do
        client:Give(weapon)
    end

    -- Chief has more health
    client:SetHealth(150)
    client:SetArmor(100)

    -- Give chief flag if they don't have it
    if !character:HasFlags("c") then
        character:GiveFlags("c")
    end

    -- Announce to faction
    for _, ply in ipairs(player.GetAll()) do
        local char = ply:GetCharacter()

        if char and char:GetFaction() == FACTION_POLICE then
            ply:ChatPrint(client:Name() .. " is now the Police Chief!")
        end
    end

    client:Notify("You are now the Police Chief")
end

-- Called when leaving
function CLASS:OnRemoved(client)
    -- Remove weapons
    for _, weapon in ipairs(self.weapons) do
        client:StripWeapon(weapon)
    end

    -- Reset health
    client:SetHealth(100)
    client:SetArmor(50)

    -- Announce
    for _, ply in ipairs(player.GetAll()) do
        local char = ply:GetCharacter()

        if char and char:GetFaction() == FACTION_POLICE then
            ply:ChatPrint(client:Name() .. " is no longer the Police Chief")
        end
    end
end

CLASS_CHIEF = CLASS.index
```

### Key Points

- **`CanSwitchTo()`**: Check multiple requirements
- **Return `false, "reason"`**: Deny with message
- **`limit = 1`**: Only one player can have this class
- **Faction-wide announcements**: Notify all faction members

## Example 3: Medical Class with Abilities

A class that grants special abilities.

### File Location

**Place in**: `schema/classes/sh_doctor.lua`

### Complete Code

```lua
CLASS.name = "Doctor"
CLASS.description = "Medical professional who can heal others"
CLASS.faction = FACTION_MEDICAL
CLASS.limit = 3
CLASS.pay = 100

-- Called when joining
function CLASS:OnSet(client)
    local character = client:GetCharacter()

    -- Give medical items
    local inventory = character:GetInventory()
    inventory:Add("item_medkit", 3)
    inventory:Add("item_bandage", 5)

    -- Give equipment
    client:SetHealth(100)
    client:SetArmor(25)

    -- Enable healing ability
    character:SetData("canHeal", true)

    client:Notify("You can now heal other players with your medical kit")
end

-- Called when leaving
function CLASS:OnRemoved(client)
    local character = client:GetCharacter()

    -- Disable healing ability
    character:SetData("canHeal", nil)
end

-- Custom ability cooldowns
CLASS.healCooldowns = CLASS.healCooldowns or {}

-- Heal function (called from item or command)
function CLASS:HealPlayer(healer, target)
    local healerChar = healer:GetCharacter()

    if !healerChar:GetData("canHeal") then
        return false, "You are not a doctor"
    end

    -- Check cooldown
    local steamID = healer:SteamID()
    local cooldown = self.healCooldowns[steamID]

    if cooldown and cooldown > CurTime() then
        local remaining = math.ceil(cooldown - CurTime())
        return false, "You must wait " .. remaining .. " seconds"
    end

    -- Check distance
    if healer:GetPos():Distance(target:GetPos()) > 150 then
        return false, "Target is too far away"
    end

    -- Heal target
    local healAmount = 50
    local newHealth = math.min(target:Health() + healAmount, target:GetMaxHealth())
    target:SetHealth(newHealth)

    -- Effects
    target:EmitSound("items/medshot4.wav")
    healer:EmitSound("items/smallmedkit1.wav")

    -- Notify
    healer:Notify("You healed " .. target:Name() .. " for " .. healAmount .. " HP")
    target:Notify(healer:Name() .. " healed you for " .. healAmount .. " HP")

    -- Set cooldown (30 seconds)
    self.healCooldowns[steamID] = CurTime() + 30

    -- Give doctor experience
    healerChar:SetData("healsPerformed", (healerChar:GetData("healsPerformed", 0) + 1))

    return true
end

CLASS_DOCTOR = CLASS.index
```

### Usage Command

```lua
-- Add this command to your schema/commands
ix.command.Add("Heal", {
    description = "Heal a nearby player (Doctor only)",
    arguments = {
        ix.type.player
    },
    OnRun = function(self, client, target)
        local class = ix.class.list[CLASS_DOCTOR]

        if class then
            local success, msg = class:HealPlayer(client, target)

            if !success then
                return msg
            end

            return "Healed " .. target:Name()
        end

        return "Doctor class not found"
    end
})
```

### Key Points

- **Character data**: Store abilities with `SetData()`
- **Cooldowns**: Prevent spam with cooldown table
- **Custom methods**: Add class-specific functions
- **Integration**: Use with commands and items

## Example 4: Military Class with Loadouts

A combat class with equipment selection.

### File Location

**Place in**: `schema/classes/sh_soldier.lua`

### Complete Code

```lua
CLASS.name = "Soldier"
CLASS.description = "Combat specialist with multiple loadouts"
CLASS.faction = FACTION_MILITARY
CLASS.limit = 8

-- Available loadouts
CLASS.loadouts = {
    assault = {
        name = "Assault",
        weapons = {"weapon_smg1", "weapon_pistol"},
        health = 100,
        armor = 75
    },
    support = {
        name = "Support",
        weapons = {"weapon_shotgun", "weapon_pistol"},
        health = 125,
        armor = 50
    },
    sniper = {
        name = "Sniper",
        weapons = {"weapon_crossbow", "weapon_pistol"},
        health = 75,
        armor = 50
    }
}

-- Called when joining
function CLASS:OnSet(client)
    local character = client:GetCharacter()

    -- Prompt for loadout selection
    if CLIENT then
        -- Show loadout menu (would be in derma file)
        -- For now, set default
        character:SetData("loadout", "assault")
    end

    -- Apply default loadout
    self:ApplyLoadout(client, "assault")

    client:Notify("Select your loadout with /selectloadout")
end

-- Apply a loadout
function CLASS:ApplyLoadout(client, loadoutName)
    local loadout = self.loadouts[loadoutName]

    if !loadout then
        return false
    end

    local character = client:GetCharacter()

    -- Remove old weapons
    for _, weapon in ipairs({"weapon_smg1", "weapon_shotgun", "weapon_crossbow", "weapon_pistol"}) do
        if client:HasWeapon(weapon) then
            client:StripWeapon(weapon)
        end
    end

    -- Give new weapons
    for _, weapon in ipairs(loadout.weapons) do
        client:Give(weapon)
    end

    -- Set stats
    client:SetHealth(loadout.health)
    client:SetArmor(loadout.armor)

    -- Save loadout
    character:SetData("loadout", loadoutName)

    client:Notify("Loadout changed to: " .. loadout.name)

    return true
end

-- Called when leaving
function CLASS:OnRemoved(client)
    -- Remove all loadout weapons
    for loadoutName, loadout in pairs(self.loadouts) do
        for _, weapon in ipairs(loadout.weapons) do
            if client:HasWeapon(weapon) then
                client:StripWeapon(weapon)
            end
        end
    end
end

-- Respawn with same loadout
function CLASS:OnPlayerSpawn(client)
    local character = client:GetCharacter()

    if character and character:GetClass() == self.index then
        local loadout = character:GetData("loadout", "assault")
        self:ApplyLoadout(client, loadout)
    end
end

CLASS_SOLDIER = CLASS.index
```

### Loadout Command

```lua
ix.command.Add("SelectLoadout", {
    description = "Select your soldier loadout",
    arguments = {
        ix.type.string  -- Loadout name
    },
    OnRun = function(self, client, loadoutName)
        local character = client:GetCharacter()

        if character:GetClass() != CLASS_SOLDIER then
            return "You must be a Soldier to use this"
        end

        local class = ix.class.list[CLASS_SOLDIER]

        if !class.loadouts[loadoutName] then
            return "Invalid loadout. Options: assault, support, sniper"
        end

        if class:ApplyLoadout(client, loadoutName) then
            return "Loadout changed to: " .. loadoutName
        else
            return "Failed to change loadout"
        end
    end
})
```

### Key Points

- **Multiple loadouts**: Define equipment sets
- **`ApplyLoadout()`**: Custom method for changing gear
- **Persistence**: Save loadout choice with character data
- **Respawn handling**: Reapply loadout on spawn

## Example 5: VIP Class with Perks

A premium class with passive benefits.

### File Location

**Place in**: `schema/classes/sh_vipguard.lua`

### Complete Code

```lua
CLASS.name = "VIP Guard"
CLASS.description = "Elite guards with special training"
CLASS.faction = FACTION_SECURITY
CLASS.limit = 2

-- Check VIP status
function CLASS:CanSwitchTo(client)
    if !client:IsUserGroup("vip") and !client:IsUserGroup("admin") then
        return false, "This class requires VIP membership"
    end

    return true
end

-- Called when joining
function CLASS:OnSet(client)
    local character = client:GetCharacter()

    -- VIP weapons
    client:Give("weapon_ar2")
    client:Give("weapon_pistol")
    client:Give("weapon_stunstick")

    -- Enhanced stats
    client:SetHealth(150)
    client:SetArmor(100)
    client:SetRunSpeed(280)
    client:SetWalkSpeed(110)

    -- VIP perks
    character:SetData("vipGuard", true)
    character:SetData("damageReduction", 0.15)  -- 15% damage reduction
    character:SetData("healthRegen", true)

    -- Give starting items
    local inventory = character:GetInventory()
    inventory:Add("item_medkit", 2)
    inventory:Add("item_armor_vest", 1)

    client:Notify("You are now a VIP Guard with special perks!")
end

-- Called when leaving
function CLASS:OnRemoved(client)
    local character = client:GetCharacter()

    -- Remove weapons
    client:StripWeapon("weapon_ar2")
    client:StripWeapon("weapon_pistol")
    client:StripWeapon("weapon_stunstick")

    -- Reset stats
    client:SetHealth(100)
    client:SetArmor(0)
    client:SetRunSpeed(240)
    client:SetWalkSpeed(100)

    -- Remove perks
    character:SetData("vipGuard", nil)
    character:SetData("damageReduction", nil)
    character:SetData("healthRegen", nil)
end

CLASS_VIPGUARD = CLASS.index
```

### Hook for Damage Reduction

```lua
-- In schema or plugin
function PLUGIN:EntityTakeDamage(target, dmgInfo)
    if !target:IsPlayer() then return end

    local character = target:GetCharacter()

    if !character then return end

    -- Check for damage reduction perk
    local reduction = character:GetData("damageReduction", 0)

    if reduction > 0 then
        local damage = dmgInfo:GetDamage()
        dmgInfo:SetDamage(damage * (1 - reduction))
    end
end
```

### Hook for Health Regen

```lua
-- In schema or plugin
function PLUGIN:Think()
    for _, client in ipairs(player.GetAll()) do
        local character = client:GetCharacter()

        if character and character:GetData("healthRegen") then
            if client:Health() < client:GetMaxHealth() then
                client:SetHealth(math.min(client:Health() + 0.1, client:GetMaxHealth()))
            end
        end
    end
end
```

### Key Points

- **VIP restrictions**: Check usergroup in `CanSwitchTo()`
- **Passive perks**: Use character data + hooks
- **Enhanced stats**: Better speed, health, armor
- **Integration**: Hooks check for class data

## ⚠️ Do NOT

```lua
-- WRONG: Don't forget faction
CLASS.name = "Officer"
-- Missing CLASS.faction!

-- WRONG: Don't modify in CanSwitchTo
function CLASS:CanSwitchTo(client)
    client:SetHealth(100)  -- NO! Just check permission
    return true
end

-- WRONG: Don't forget to clean up
function CLASS:OnSet(client)
    client:Give("weapon_pistol")
end
-- Missing OnRemoved to strip weapon!

-- WRONG: Don't use client functions on server
function CLASS:OnSet(client)
    LocalPlayer():ChatPrint("Joined")  -- LocalPlayer() doesn't exist on server!
end
```

## Best Practices

### ✅ DO

- Set both `OnSet()` and `OnRemoved()`
- Strip weapons in `OnRemoved()` that were given in `OnSet()`
- Use `CanSwitchTo()` for joining requirements
- Store class data with `character:SetData()`
- Clean up timers and data in `OnRemoved()`
- Use `limit` to prevent too many of one class
- Return descriptive error messages from `CanSwitchTo()`
- Check `IsValid()` before using entities
- Use `character:GetClass()` to check current class

### ❌ DON'T

- Don't forget to set `CLASS.faction`
- Don't modify player state in `CanSwitchTo()`
- Don't forget to strip weapons in `OnRemoved()`
- Don't forget realm checks for CLIENT code
- Don't create infinite loops in class switching
- Don't give too many weapons
- Don't forget to validate faction index exists

## Advanced Tips

### Class Commands

```lua
ix.command.Add("BecomeClass", {
    description = "Join a class",
    arguments = {
        ix.type.string  -- Class name
    },
    OnRun = function(self, client, className)
        local character = client:GetCharacter()

        -- Find class by name
        local targetClass = nil
        for _, class in pairs(ix.class.list) do
            if string.lower(class.name) == string.lower(className) then
                targetClass = class
                break
            end
        end

        if !targetClass then
            return "Class not found"
        end

        -- Try to join
        if character:JoinClass(targetClass.index) then
            return "Joined " .. targetClass.name
        else
            return "Cannot join that class"
        end
    end
})
```

### Class Progression

```lua
-- Track class time
function PLUGIN:PlayerLoadedCharacter(client, character)
    local class = character:GetClass()

    if class and class != 0 then
        local uniqueID = "ixClassTime_" .. client:SteamID()

        timer.Create(uniqueID, 60, 0, function()
            if IsValid(client) then
                local char = client:GetCharacter()

                if char and char:GetClass() == class then
                    local time = char:GetData("classTime_" .. class, 0)
                    char:SetData("classTime_" .. class, time + 1)
                end
            end
        end)
    end
end
```

## See Also

- [Class System](../systems/classes.md) - Complete class documentation
- [Faction System](../systems/factions.md) - Working with factions
- [Character System](../systems/character.md) - Character functions
- [Command System](../systems/commands.md) - Creating commands
- Source: `gamemode/core/libs/sh_class.lua` - Class implementation
- Source: `gamemode/core/meta/sh_character.lua` - Character methods
