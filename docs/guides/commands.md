# Command System Tutorial

> **Reference**: `gamemode/core/libs/sh_command.lua:38`

Complete guide to creating custom commands in Helix, from basic chat commands to complex administrative tools.

## ⚠️ Important: Use Helix's Command System

**Always use `ix.command.Add()`** rather than creating console commands or custom chat listeners. The framework provides:
- Automatic argument validation and type checking
- Built-in permission system via CAMI
- Automatic help text generation
- Tab completion for player names
- Support for multiple command prefixes (`/`, `@`, `!`)

## Prerequisites

- Basic understanding of Lua functions
- Schema or plugin set up
- Familiarity with Helix structure

## Command Basics

### Anatomy of a Command

```lua
ix.command.Add("CommandName", {
    description = "What it does",
    arguments = {ix.type.player, ix.type.number},  -- Optional
    adminOnly = false,  -- Optional

    OnRun = function(self, client, target, amount)
        -- Your code here
        return "Feedback message"
    end
})
```

### Command Execution

Players can run commands three ways:
- `/command args` - Normal (shows feedback in chat)
- `@command args` - Silent (no chat feedback)
- `!command args` - Alternative prefix

### Where to Register Commands

**✅ Correct location:**
```lua
// In plugin
function PLUGIN:InitializedPlugins()
    ix.command.Add("MyCommand", {...})
end

// In schema
function Schema:InitializedPlugins()
    ix.command.Add("MyCommand", {...})
end
```

**❌ Wrong location:**
```lua
// At top of file - TOO EARLY!
ix.command.Add("MyCommand", {...})

PLUGIN.name = "My Plugin"
```

## Part 1: Simple Commands

### Example 1: No Arguments

```lua
ix.command.Add("ServerTime", {
    description = "Shows the current server time",
    OnRun = function(self, client)
        local time = os.date("%I:%M %p")
        return "Server time: " .. time
    end
})
```

**Usage**: `/servertime`
**Output**: `Server time: 03:45 PM`

### Example 2: With String Argument

```lua
ix.command.Add("Broadcast", {
    description = "Send a message to all players",
    adminOnly = true,
    arguments = ix.type.text,  -- All remaining text

    OnRun = function(self, client, message)
        for _, ply in ipairs(player.GetAll()) do
            ply:ChatPrint("[BROADCAST] " .. message)
        end
        return "Broadcast sent!"
    end
})
```

**Usage**: `/broadcast Server will restart in 5 minutes`

### Example 3: Multiple Arguments

```lua
ix.command.Add("GiveMoney", {
    description = "Give money to a player",
    adminOnly = true,
    arguments = {
        ix.type.character,  -- Target character
        ix.type.number      -- Amount
    },

    OnRun = function(self, client, target, amount)
        -- Validate amount
        if amount <= 0 then
            return "Amount must be positive!"
        end

        if amount > 10000 then
            return "Maximum amount is 10000!"
        end

        -- Give money
        target:GiveMoney(amount)

        -- Notify target
        local targetClient = target:GetPlayer()
        targetClient:Notify("You received $" .. amount)

        return "Gave $" .. amount .. " to " .. target:GetName()
    end
})
```

**Usage**: `/givemoney "John Doe" 500`

## Part 2: Argument Types

### String vs Text

```lua
-- ix.type.string = ONE word only
arguments = {ix.type.string}
-- Usage: /command word
-- Invalid: /command multiple words

-- ix.type.text = ALL remaining text
arguments = {ix.type.text}
-- Usage: /command this is all one argument
-- Gets: "this is all one argument"
```

**Example: Kick Command**
```lua
ix.command.Add("Kick", {
    description = "Kick a player",
    superAdminOnly = true,
    arguments = {
        ix.type.player,  -- Target
        ix.type.text     -- Reason (all remaining text)
    },

    OnRun = function(self, client, target, reason)
        reason = reason or "No reason given"

        -- Log to console
        print(string.format("[KICK] %s kicked %s: %s",
            client:Name(), target:Name(), reason))

        -- Kick player
        target:Kick(reason)

        return string.format("Kicked %s: %s", target:Name(), reason)
    end
})
```

**Usage**: `/kick "Bad Player" Breaking server rules`

### Player vs Character

```lua
-- ix.type.player = Returns player entity
arguments = {ix.type.player}
OnRun = function(self, client, target)
    print(target:Name())  -- Player name
    print(target:SteamID())  -- Steam ID
end

-- ix.type.character = Returns character object
arguments = {ix.type.character}
OnRun = function(self, client, targetChar)
    print(targetChar:GetName())  -- Character name
    print(targetChar:GetMoney())  -- Character money
end
```

### Number Type

```lua
ix.command.Add("SetHealth", {
    description = "Set a player's health",
    adminOnly = true,
    arguments = {
        ix.type.player,
        ix.type.number
    },

    OnRun = function(self, client, target, health)
        -- Validate number
        if health < 1 or health > 100 then
            return "Health must be between 1 and 100!"
        end

        target:SetHealth(health)
        return string.format("Set %s's health to %d", target:Name(), health)
    end
})
```

**Usage**: `/sethealth john 50`

### Optional Arguments

```lua
ix.command.Add("Heal", {
    description = "Heal yourself or another player",
    adminOnly = true,
    arguments = {
        bit.bor(ix.type.player, ix.type.optional)  -- Optional target
    },

    OnRun = function(self, client, target)
        -- If no target specified, heal self
        target = target or client

        target:SetHealth(target:GetMaxHealth())

        if target == client then
            return "You healed yourself"
        else
            return "You healed " .. target:Name()
        end
    end
})
```

**Usage**:
- `/heal` - Heals yourself
- `/heal john` - Heals John

## Part 3: Permission Systems

### Admin-Only Commands

```lua
ix.command.Add("Noclip", {
    description = "Toggle noclip",
    adminOnly = true,  -- Requires admin

    OnRun = function(self, client)
        local mode = client:GetMoveType()

        if mode == MOVETYPE_NOCLIP then
            client:SetMoveType(MOVETYPE_WALK)
            return "Noclip disabled"
        else
            client:SetMoveType(MOVETYPE_NOCLIP)
            return "Noclip enabled"
        end
    end
})
```

### SuperAdmin-Only

```lua
ix.command.Add("RestartServer", {
    description = "Restart the server",
    superAdminOnly = true,  -- Only superadmins

    OnRun = function(self, client)
        -- Warn all players
        for _, ply in ipairs(player.GetAll()) do
            ply:ChatPrint("[SERVER] Restarting in 30 seconds!")
        end

        -- Restart after delay
        timer.Simple(30, function()
            RunConsoleCommand("changelevel", game.GetMap())
        end)

        return "Server restart initiated"
    end
})
```

### Custom Permission Check

```lua
ix.command.Add("ManageVendors", {
    description = "Open vendor management panel",

    -- Custom access check
    OnCheckAccess = function(self, client)
        local char = client:GetCharacter()

        -- Check for specific flag
        if not char:HasFlags("v") then
            return false
        end

        -- Check for faction
        if char:GetFaction() ~= FACTION_MERCHANT then
            return false
        end

        return true
    end,

    OnRun = function(self, client)
        -- Open vendor menu
        net.Start("OpenVendorPanel")
        net.Send(client)

        return "Opening vendor management"
    end
})
```

### Flag-Based Permissions

```lua
ix.command.Add("Arrest", {
    description = "Arrest a player",

    OnCheckAccess = function(self, client)
        local character = client:GetCharacter()

        -- Require police flag
        return character and character:HasFlags("p")
    end,

    arguments = {
        ix.type.character,
        ix.type.number,  -- Time in seconds
        ix.type.text     -- Reason
    },

    OnRun = function(self, client, target, time, reason)
        -- Arrest logic
        target:SetData("arrested", true)
        target:SetData("arrestTime", time)
        target:SetData("arrestReason", reason)

        local targetClient = target:GetPlayer()
        targetClient:Freeze(true)

        return string.format("Arrested %s for %d seconds: %s",
            target:GetName(), time, reason)
    end
})
```

## Part 4: Advanced Commands

### Command with Cooldown

```lua
-- Store last use times
local lastUse = {}

ix.command.Add("Heal", {
    description = "Heal yourself (5 minute cooldown)",

    OnRun = function(self, client)
        local steamID = client:SteamID()
        local now = CurTime()
        local cooldown = 300  -- 5 minutes

        -- Check cooldown
        if lastUse[steamID] and (now - lastUse[steamID]) < cooldown then
            local remaining = math.ceil(cooldown - (now - lastUse[steamID]))
            return string.format("Wait %d seconds before using again!", remaining)
        end

        -- Heal player
        client:SetHealth(client:GetMaxHealth())

        -- Update last use
        lastUse[steamID] = now

        return "You healed yourself"
    end
})
```

### Command with Confirmation

```lua
-- Store pending confirmations
local pendingWipes = {}

ix.command.Add("WipeInventory", {
    description = "Wipe your inventory (requires confirmation)",

    arguments = {
        bit.bor(ix.type.string, ix.type.optional)
    },

    OnRun = function(self, client, confirm)
        local steamID = client:SteamID()

        if confirm == "confirm" then
            -- Check if confirmation pending
            if not pendingWipes[steamID] then
                return "No wipe pending! Use /wipeinventory first"
            end

            -- Clear confirmation
            pendingWipes[steamID] = nil

            -- Wipe inventory
            local character = client:GetCharacter()
            local inventory = character:GetInventory()

            for _, item in pairs(inventory:GetItems()) do
                item:Remove()
            end

            return "Your inventory has been wiped"
        else
            -- Request confirmation
            pendingWipes[steamID] = true

            -- Clear after 30 seconds
            timer.Simple(30, function()
                pendingWipes[steamID] = nil
            end)

            return "Type /wipeinventory confirm to wipe your inventory"
        end
    end
})
```

### Command with Multiple Actions

```lua
ix.command.Add("CharManage", {
    description = "Character management command",
    adminOnly = true,
    arguments = {
        ix.type.character,
        ix.type.string,  -- Action
        bit.bor(ix.type.text, ix.type.optional)  -- Value
    },

    OnRun = function(self, client, target, action, value)
        action = action:lower()

        if action == "setname" then
            if not value then
                return "Usage: /charmanage player setname <newname>"
            end

            local oldName = target:GetName()
            target:SetName(value)

            return string.format("Changed %s's name to %s", oldName, value)

        elseif action == "resetmoney" then
            local oldMoney = target:GetMoney()
            target:SetMoney(0)

            return string.format("Reset %s's money ($%d to $0)", target:GetName(), oldMoney)

        elseif action == "heal" then
            local targetClient = target:GetPlayer()
            targetClient:SetHealth(targetClient:GetMaxHealth())

            return string.format("Healed %s", target:GetName())

        elseif action == "info" then
            local info = string.format(
                "%s | Money: $%d | Faction: %s",
                target:GetName(),
                target:GetMoney(),
                ix.faction.Get(target:GetFaction()).name
            )

            return info

        else
            return "Unknown action! Available: setname, resetmoney, heal, info"
        end
    end
})
```

**Usage**:
- `/charmanage john setname "John Smith"`
- `/charmanage john resetmoney`
- `/charmanage john heal`
- `/charmanage john info`

## Best Practices

### ✅ DO

```lua
// Register in InitializedPlugins hook
function PLUGIN:InitializedPlugins()
    ix.command.Add("MyCommand", {...})
end

// Validate all inputs
OnRun = function(self, client, amount)
    if amount <= 0 then
        return "Amount must be positive!"
    end
    -- rest of code
end

// Use argument types for validation
arguments = {ix.type.number, ix.type.player}

// Return helpful feedback
return "Successfully gave 100 money to John"

// Check permissions properly
OnCheckAccess = function(self, client)
    return client:GetCharacter():HasFlags("a")
end

// Use localized strings
return client:Notify("commandSuccess")
```

### ❌ DON'T

```lua
// Don't register at top level
ix.command.Add("MyCommand", {...})  // Too early!

// Don't skip validation
OnRun = function(self, client, amount)
    target:GiveMoney(amount)  // What if amount is negative?
end

// Don't forget return values
OnRun = function(self, client)
    client:SetHealth(100)
    // No return = no feedback!
end

// Don't hardcode permissions
OnRun = function(self, client)
    if client:Name() == "Admin" then  // BAD!
        // ...
    end
end

// Don't use player names directly
arguments = {ix.type.string}  // Use ix.type.player instead!
OnRun = function(self, client, name)
    local target = player.GetByName(name)  // Unreliable!
end
```

## Testing Commands

```lua
// Test in server console
lua_run ix.command.Run(Entity(1), "mycommand arg1 arg2")

// Check if command exists
lua_run PrintTable(ix.command.list)

// Check command structure
lua_run PrintTable(ix.command.list.mycommand)

// Test permissions
lua_run print(ix.command.list.mycommand:OnCheckAccess(Entity(1)))
```

## Troubleshooting

### Command not found

**Cause**: Not registered or wrong name

**Fix**:
```lua
// Check registered commands
lua_run PrintTable(ix.command.list)

// Ensure registered in hook
function PLUGIN:InitializedPlugins()
    ix.command.Add("CommandName", {...})
end
```

### Arguments not working

**Cause**: Wrong argument type or structure

**Fix**:
```lua
// ✅ CORRECT
arguments = {ix.type.player, ix.type.number}

// ❌ WRONG
arguments = {player, number}
arguments = ix.type.player, ix.type.number  // Missing {}
```

### Permission denied

**Cause**: `OnCheckAccess` returning false

**Fix**:
```lua
// Debug access check
OnCheckAccess = function(self, client)
    local result = client:IsAdmin()
    print("Access check for", client:Name(), "=", result)
    return result
end
```

### No feedback shown

**Cause**: Not returning a value from `OnRun`

**Fix**:
```lua
// ❌ WRONG
OnRun = function(self, client)
    client:SetHealth(100)
    // No return!
end

// ✅ CORRECT
OnRun = function(self, client)
    client:SetHealth(100)
    return "Health set to 100"
end
```

## Complete Example: Ban System

Here's a complete command system for character banning:

```lua
function PLUGIN:InitializedPlugins()
    -- Ban command
    ix.command.Add("CharBan", {
        description = "Ban a character from the server",
        superAdminOnly = true,
        arguments = {
            ix.type.character,
            ix.type.number,  -- Duration in minutes
            ix.type.text     -- Reason
        },

        OnRun = function(self, client, target, duration, reason)
            -- Calculate unban time
            local unbanTime = os.time() + (duration * 60)

            -- Store ban data
            target:SetData("banned", true)
            target:SetData("banTime", os.time())
            target:SetData("banDuration", duration)
            target:SetData("banReason", reason)
            target:SetData("bannedBy", client:SteamID())
            target:SetData("unbanTime", unbanTime)

            -- Kick player
            local targetClient = target:GetPlayer()
            targetClient:Kick("Banned: " .. reason)

            -- Log
            ix.log.Add(client, "charBan", target:GetName(), duration, reason)

            return string.format("Banned %s for %d minutes: %s",
                target:GetName(), duration, reason)
        end
    })

    -- Unban command
    ix.command.Add("CharUnban", {
        description = "Unban a character",
        superAdminOnly = true,
        arguments = {ix.type.character},

        OnRun = function(self, client, target)
            if not target:GetData("banned") then
                return target:GetName() .. " is not banned"
            end

            -- Remove ban
            target:SetData("banned", nil)
            target:SetData("banTime", nil)
            target:SetData("banDuration", nil)
            target:SetData("banReason", nil)
            target:SetData("bannedBy", nil)
            target:SetData("unbanTime", nil)

            -- Log
            ix.log.Add(client, "charUnban", target:GetName())

            return "Unbanned " .. target:GetName()
        end
    })

    -- Check ban status
    ix.command.Add("CharBanInfo", {
        description = "Check a character's ban status",
        adminOnly = true,
        arguments = {ix.type.character},

        OnRun = function(self, client, target)
            if not target:GetData("banned") then
                return target:GetName() .. " is not banned"
            end

            local banTime = target:GetData("banTime")
            local duration = target:GetData("banDuration")
            local reason = target:GetData("banReason")
            local unbanTime = target:GetData("unbanTime")

            local remaining = math.max(0, unbanTime - os.time())
            local remainingMin = math.ceil(remaining / 60)

            return string.format(
                "%s is banned | Reason: %s | Time remaining: %d minutes",
                target:GetName(), reason, remainingMin
            )
        end
    })
end
```

## See Also

- [Commands System Documentation](../systems/commands.md)
- [First Plugin Tutorial](first-plugin.md)
- [Hooks Guide](hooks.md)
- Source: `gamemode/core/libs/sh_command.lua`
