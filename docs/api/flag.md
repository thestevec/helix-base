# Flag API (ix.flag)

> **Reference**: `gamemode/core/libs/sh_flag.lua`

The flag API provides character-based permissions using single alphanumeric characters. Flags grant abilities like physgun access, prop spawning, or custom permissions without requiring admin status.

## ⚠️ Important: Use Built-in Helix Flag System

**Always use Helix's built-in flag system** for character permissions. The framework automatically provides:
- Simple alphanumeric flag characters (a-z, A-Z, 0-9)
- Server-side validation
- Automatic callback triggering
- Character-specific permissions
- Flag display in Help menu
- Network synchronization

## Core Concepts

### What are Flags?

Flags are single-character permissions (e.g., "p" for physgun, "t" for toolgun):
- Assigned per-character (not per-player)
- Single alphanumeric characters
- Have descriptions shown in Help menu
- Can trigger callbacks when given/removed
- Validated server-side

### Default Flags

```
p - Physgun access
t - Toolgun access
e - Spawn props
c - Spawn chairs
C - Spawn vehicles
r - Spawn ragdolls
n - Spawn NPCs
```

## Library Tables

### ix.flag.list

**Reference**: `gamemode/core/libs/sh_flag.lua:31`

**Realm**: Shared

```lua
-- Check flag info
local flagInfo = ix.flag.list["p"]
print(flagInfo.description)  -- "Access to the physgun."

-- List all flags
for flag, info in pairs(ix.flag.list) do
    print(flag, info.description)
end
```

## Library Functions

### ix.flag.Add

**Reference**: `gamemode/core/libs/sh_flag.lua:38`

**Realm**: Shared

```lua
ix.flag.Add(flag, description, callback)
```

Registers a new flag.

**Parameters**:
- `flag` (string) - Single alphanumeric character
- `description` (string) - Description for Help menu
- `callback` (function, optional) - Called when flag given/removed: `callback(client, isGiven)`

**Example**:
```lua
-- Simple flag
ix.flag.Add("a", "Admin commands access")

-- Flag with callback
ix.flag.Add("v", "VIP benefits", function(client, isGiven)
    if isGiven then
        client:ChatPrint("VIP benefits activated!")
        client:SetMaxHealth(150)
    else
        client:ChatPrint("VIP benefits removed")
        client:SetMaxHealth(100)
    end
end)

-- Weapon flag
ix.flag.Add("w", "Access to advanced weapons", function(client, isGiven)
    if isGiven then
        client:Give("weapon_smg")
    else
        client:StripWeapon("weapon_smg")
    end
end)
```

## Character Methods

### character:HasFlags

**Reference**: `gamemode/core/libs/sh_flag.lua:160`

**Realm**: Shared

```lua
local has = character:HasFlags(flags)
```

Checks if character has any of the specified flags.

**Parameters**:
- `flags` (string) - Flag(s) to check (e.g., "p", "pt", "abc")

**Returns**: (bool) True if has ANY of the flags

**Example**:
```lua
-- Check single flag
if character:HasFlags("p") then
    client:Give("weapon_physgun")
end

-- Check multiple flags (OR logic)
if character:HasFlags("pt") then
    -- Has 'p' OR 't' or both
    print("Has physgun or toolgun access")
end

-- Common pattern
if not character:HasFlags("a") then
    return "You need admin flag"
end
```

### character:GetFlags

**Reference**: `gamemode/core/libs/sh_flag.lua:152`

**Realm**: Shared

```lua
local flags = character:GetFlags()
```

Gets all flags as a string.

**Returns**: (string) All flags character has

**Example**:
```lua
local flags = character:GetFlags()
print("Flags:", flags)  -- "pet" (has p, e, t)

-- Iterate individual flags
for i = 1, #flags do
    local flag = flags[i]
    print("Has flag:", flag)
end
```

### character:GiveFlags

**Reference**: `gamemode/core/libs/sh_flag.lua:92`

**Realm**: Server

```lua
character:GiveFlags(flags)
```

Gives flag(s) to character.

**Parameters**:
- `flags` (string) - Flag(s) to give (e.g., "p", "pet")

**Example**:
```lua
-- Give single flag
character:GiveFlags("p")

-- Give multiple flags
character:GiveFlags("pet")  -- Gives p, e, and t

-- Admin command
ix.command.Add("GiveFlag", {
    adminOnly = true,
    arguments = {ix.type.character, ix.type.string},
    OnRun = function(self, client, target, flags)
        target:GiveFlags(flags)
        return "Gave flags '" .. flags .. "' to " .. target:GetName()
    end
})
```

**Note**: Triggers callbacks for each flag given.

### character:TakeFlags

**Reference**: `gamemode/core/libs/sh_flag.lua:124`

**Realm**: Server

```lua
character:TakeFlags(flags)
```

Removes flag(s) from character.

**Parameters**:
- `flags` (string) - Flag(s) to remove

**Example**:
```lua
-- Remove single flag
character:TakeFlags("p")

-- Remove multiple flags
character:TakeFlags("pet")

-- Revoke on death
hook.Add("PlayerDeath", "RevokeFlagsOnDeath", function(victim)
    local char = victim:GetCharacter()
    if char then
        char:TakeFlags("pt")  -- Remove physgun and toolgun
    end
end)
```

**Note**: Triggers callbacks for each flag removed.

### character:SetFlags

**Reference**: `gamemode/core/libs/sh_flag.lua:82`

**Realm**: Server

```lua
character:SetFlags(flags)
```

Sets flags, replacing all existing flags.

**Parameters**:
- `flags` (string) - New flag string

**Example**:
```lua
-- Reset all flags
character:SetFlags("")

-- Set specific flags only
character:SetFlags("pet")

-- ⚠️ WARNING: Overwrites existing flags
character:SetFlags("p")  -- Removes all other flags!
```

**⚠️ Warning**: Use `GiveFlags/TakeFlags` instead unless you need to replace all flags.

## Complete Examples

### VIP System

```lua
ix.flag.Add("v", "VIP member benefits", function(client, isGiven)
    if isGiven then
        -- Grant VIP benefits
        client:SetMaxHealth(150)
        client:SetHealth(150)
        client:SetWalkSpeed(120)
        client:SetRunSpeed(250)

        client:Notify("VIP benefits activated!")
    else
        -- Remove VIP benefits
        client:SetMaxHealth(100)
        client:SetWalkSpeed(100)
        client:SetRunSpeed(200)

        client:Notify("VIP benefits removed")
    end
end)

-- Command to give VIP
ix.command.Add("MakeVIP", {
    adminOnly = true,
    arguments = ix.type.character,
    OnRun = function(self, client, target)
        target:GiveFlags("v")
        return target:GetName() .. " is now VIP"
    end
})
```

### Door Access Flag

```lua
ix.flag.Add("d", "Access to secure doors")

-- Check flag when using door
hook.Add("PlayerUse", "DoorAccess", function(client, entity)
    if entity:GetClass() == "func_door" and entity:GetNWBool("secure") then
        local char = client:GetCharacter()

        if not char or not char:HasFlags("d") then
            client:Notify("This door requires security clearance")
            return false
        end
    end
end)
```

### Rank-Based Flags

```lua
-- Automatically give flags based on rank
hook.Add("PlayerLoadedCharacter", "RankFlags", function(client, character)
    local faction = character:GetFaction()
    local class = character:GetClass()

    -- Police get toolgun and physgun
    if faction == FACTION_POLICE then
        character:GiveFlags("pt")

        -- Police Chief gets additional flags
        if class == CLASS_CHIEF then
            character:GiveFlags("ae")  -- Admin and prop spawn
        end
    end
end)
```

### Temporary Flag

```lua
-- Grant temporary flag
function GrantTemporaryFlag(character, flag, duration)
    local client = character:GetPlayer()

    -- Give flag
    character:GiveFlags(flag)
    client:Notify("Temporary access granted for " .. duration .. " seconds")

    -- Remove after duration
    timer.Simple(duration, function()
        if character and IsValid(client) then
            character:TakeFlags(flag)
            client:Notify("Temporary access expired")
        end
    end)
end

-- Usage
GrantTemporaryFlag(character, "p", 300)  -- 5 minutes of physgun
```

## Best Practices

### ✅ DO

- Use single alphanumeric characters
- Provide clear descriptions
- Use callbacks for temporary effects
- Check flags server-side
- Give meaningful flag letters (p = physgun, t = toolgun)
- Use uppercase for special variants (c = chairs, C = vehicles)

### ❌ DON'T

- Don't use multi-character flags
- Don't bypass flag checks
- Don't use SetFlags unless replacing all flags
- Don't forget to remove callbacks (weapons, etc.)
- Don't confuse flags with admin permissions
- Don't use flags for admin-only actions

## Common Patterns

### Flag Requirement Check

```lua
ix.command.Add("SecureCommand", {
    description = "Requires security flag",
    OnRun = function(self, client)
        local char = client:GetCharacter()

        if not char:HasFlags("s") then
            return "Requires security clearance (flag: s)"
        end

        -- Execute command
    end
})
```

### Multiple Flag Requirement (AND)

```lua
-- Requires BOTH flags
if character:HasFlags("a") and character:HasFlags("s") then
    -- Has both admin AND security flags
end

-- Or check the string
local flags = character:GetFlags()
if flags:find("a") and flags:find("s") then
    -- Same as above
end
```

### Flag Menu

```lua
-- Show flags in F1 menu
hook.Add("PopulateHelpMenu", "ShowFlags", function(tabs)
    local flags = LocalPlayer():GetCharacter():GetFlags()

    tabs["flags"] = function(container)
        for i = 1, #flags do
            local flag = flags[i]
            local info = ix.flag.list[flag]

            if info then
                container:Add("DLabel"):SetText(flag .. " - " .. info.description)
            end
        end
    end
end)
```

## Common Issues

### Callback Not Triggering

**Cause**: Flag registered only on one realm.

**Fix**: Register flags shared:
```lua
-- In shared file (sh_plugin.lua)
if CLIENT then
    -- Client code
end

ix.flag.Add("x", "Description", function(client, isGiven)
    -- This runs SERVER-SIDE only
end)
```

### Flags Not Persisting

**Cause**: Not using Give/Take/SetFlags methods.

**Fix**: Always use character methods:
```lua
-- WRONG
character:SetData("f", "pet")

-- CORRECT
character:GiveFlags("pet")
```

## Related Hooks

### CharacterHasFlags

```lua
hook.Add("CharacterHasFlags", "CustomCheck", function(character, flags)
    -- Override flag check
    if character:GetData("admin") then
        return true  -- Admins have all flags
    end
end)
```

## See Also

- [Character API](character.md) - Character flag methods
- [Command API](command.md) - Flag-restricted commands
- [Faction API](faction.md) - Faction-based flag assignment
- Source: `gamemode/core/libs/sh_flag.lua`
