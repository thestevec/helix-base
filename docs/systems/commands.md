# Command System

The Helix command system provides a robust way to create chat-based commands with type validation, permission checking, and automatic help generation.

## Overview

Commands in Helix are registered using `ix.command.Add()` and can be executed by typing in chat:
- `/command` - Standard command prefix
- `!command` - Alternative prefix
- `@command` - Silent command (no feedback in chat)

### Features

- **Type validation**: Automatic argument type checking
- **Permission system**: Admin-only or flag-based restrictions
- **Flexible arguments**: Required and optional parameters
- **Auto-completion**: Tab completion for player names
- **Help system**: Automatic command help generation
- **Cooldowns**: Built-in rate limiting

## Creating Commands

### Basic Command

```lua
ix.command.Add("Hello", {
    description = "Prints a greeting",
    OnRun = function(self, client)
        return "Hello, " .. client:Name() .. "!"
    end
})

-- Usage: /hello
-- Output: "Hello, PlayerName!"
```

### Command with Arguments

```lua
ix.command.Add("Give", {
    description = "Give an item to a player",
    arguments = {
        ix.type.player,    -- First argument: target player
        ix.type.string     -- Second argument: item name
    },
    OnRun = function(self, client, target, itemName)
        local inventory = target:GetCharacter():GetInventory()
        inventory:Add(itemName)
        return "Gave " .. itemName .. " to " .. target:Name()
    end
})

-- Usage: /give "John Doe" item_medkit
-- Output: "Gave item_medkit to John Doe"
```

### Admin-Only Command

```lua
ix.command.Add("Kick", {
    description = "Kick a player from the server",
    adminOnly = true,
    arguments = {
        ix.type.player,
        ix.type.text       -- Reason (rest of text)
    },
    OnRun = function(self, client, target, reason)
        target:Kick(reason)
        return "Kicked " .. target:Name() .. ": " .. reason
    end
})
```

## Command Structure

### Required Fields

```lua
ix.command.Add("CommandName", {
    description = "What the command does",    -- Required
    OnRun = function(self, client, ...)       -- Required
        -- Command logic here
        return "Feedback message"
    end
})
```

### Optional Fields

```lua
ix.command.Add("CommandName", {
    description = "Description",
    superAdminOnly = false,      -- Only superadmins can use
    adminOnly = false,           -- Only admins can use
    privilege = "Manage Players", -- CAMI privilege required
    arguments = {},              -- Argument types (see below)
    alias = {"cmd", "command"},  -- Alternative names
    OnCheckAccess = function(self, client)  -- Custom permission check
        return client:IsUserGroup("moderator")
    end,
    OnRun = function(self, client, ...)
        -- Command code
        return "Result message"
    end
})
```

## Argument Types

### Available Types

```lua
ix.type.string      -- Single word
ix.type.text        -- Rest of text (all remaining words)
ix.type.number      -- Any number (integer or decimal)
ix.type.player      -- Player name (with tab completion)
ix.type.steamid     -- Steam ID (STEAM_X:Y:Z)
ix.type.character   -- Character name
ix.type.bool        -- Boolean (true/false, 1/0, yes/no)
ix.type.color       -- Color (r,g,b or r,g,b,a)
ix.type.vector      -- Vector (x,y,z)

-- Modifiers
ix.type.optional    -- Makes argument optional
ix.type.array       -- Array of values
```

### Type Examples

```lua
-- String (single word)
arguments = {ix.type.string}
-- Usage: /command word

-- Text (multiple words, rest of input)
arguments = {ix.type.text}
-- Usage: /command this is all one argument

-- Number
arguments = {ix.type.number}
-- Usage: /command 42 or /command 3.14

-- Player (with auto-completion)
arguments = {ix.type.player}
-- Usage: /command "John Doe" or /command john

-- Multiple arguments
arguments = {
    ix.type.player,    -- First: target player
    ix.type.number,    -- Second: amount
    ix.type.text       -- Third: reason (rest)
}
-- Usage: /command "John Doe" 100 giving money for quest
```

### Optional Arguments

```lua
ix.command.Add("Teleport", {
    description = "Teleport to a player or to coordinates",
    arguments = {
        bit.bor(ix.type.player, ix.type.optional),  -- Optional player
        bit.bor(ix.type.vector, ix.type.optional)   -- Optional position
    },
    OnRun = function(self, client, target, position)
        if IsValid(target) then
            -- Teleport to player
            client:SetPos(target:GetPos())
            return "Teleported to " .. target:Name()
        elseif position then
            -- Teleport to position
            client:SetPos(position)
            return "Teleported to " .. tostring(position)
        else
            return "Must specify player or position"
        end
    end
})

-- Usage: /teleport "John Doe"
-- Usage: /teleport 0,0,0
-- Usage: /teleport (requires at least one argument)
```

### Array Arguments

```lua
ix.command.Add("GiveMultiple", {
    description = "Give items to multiple players",
    arguments = {
        bit.bor(ix.type.player, ix.type.array),  -- Multiple players
        ix.type.string                            -- Item name
    },
    OnRun = function(self, client, targets, itemName)
        local count = 0
        for _, target in ipairs(targets) do
            target:GetCharacter():GetInventory():Add(itemName)
            count = count + 1
        end
        return "Gave " .. itemName .. " to " .. count .. " players"
    end
})

-- Usage: /givemultiple "John Doe,Jane Smith,Bob" item_medkit
```

## Command Logic

### OnRun Function

```lua
OnRun = function(self, client, ...arguments)
    -- self: The command table
    -- client: The player who executed the command
    -- ...arguments: Validated arguments based on types

    -- Do command logic here

    -- Return string for feedback
    return "Command executed successfully"

    -- Return false to show error
    -- return false

    -- Return nothing for no feedback
    -- return
end
```

### Return Values

```lua
OnRun = function(self, client)
    -- Success message
    return "Success!"

    -- Error (shows as red notification)
    return false

    -- Error with custom message
    return "@commandError", "Custom error message"

    -- Localized message
    return "@commandSuccess"

    -- No feedback
    return
end
```

### Accessing Command Data

```lua
OnRun = function(self, client, target, amount)
    print(self.name)           -- "CommandName"
    print(self.description)    -- "Description"
    print(self.uniqueID)       -- "commandname"

    -- Use client to check permissions
    if not client:IsAdmin() then
        return "No permission"
    end

    -- Process arguments
    local char = target:GetCharacter()
    char:GiveMoney(amount)

    return "Gave $" .. amount .. " to " .. target:Name()
end
```

## Permission Checking

### Admin Only

```lua
ix.command.Add("Admin", {
    description = "Admin command",
    adminOnly = true,  -- Requires client:IsAdmin()
    OnRun = function(self, client)
        return "Admin command"
    end
})
```

### Super Admin Only

```lua
ix.command.Add("SuperAdmin", {
    description = "Super admin command",
    superAdminOnly = true,  -- Requires client:IsSuperAdmin()
    OnRun = function(self, client)
        return "Super admin command"
    end
})
```

### Custom Permission Check

```lua
ix.command.Add("Moderator", {
    description = "Moderator command",
    OnCheckAccess = function(self, client)
        -- Return true to allow, false to deny
        return client:IsUserGroup("moderator") or client:IsAdmin()
    end,
    OnRun = function(self, client)
        return "Moderator command"
    end
})
```

### Flag-Based Permissions

```lua
ix.command.Add("Vendor", {
    description = "Access vendor menu",
    OnCheckAccess = function(self, client)
        local char = client:GetCharacter()
        return char and char:HasFlags("v")  -- Requires 'v' flag
    end,
    OnRun = function(self, client)
        -- Open vendor menu
    end
})
```

### CAMI Privilege

```lua
ix.command.Add("Ban", {
    description = "Ban a player",
    privilege = "Ban",  -- CAMI privilege required
    arguments = {ix.type.player, ix.type.text},
    OnRun = function(self, client, target, reason)
        -- Ban logic
    end
})
```

## Advanced Features

### Command Aliases

```lua
ix.command.Add("Teleport", {
    description = "Teleport to a player",
    alias = {"tp", "goto"},  -- Can use /tp or /goto
    arguments = {ix.type.player},
    OnRun = function(self, client, target)
        client:SetPos(target:GetPos())
        return "Teleported to " .. target:Name()
    end
})

-- All work:
-- /teleport "John Doe"
-- /tp "John Doe"
-- /goto "John Doe"
```

### Silent Commands

```lua
-- Silent commands don't show feedback in chat
-- Prefix with @ instead of /

-- /broadcast hello     - Everyone sees "hello"
-- @broadcast hello     - Only shows to admins
```

### Command Cooldowns

```lua
PLUGIN.commandCooldowns = PLUGIN.commandCooldowns or {}

ix.command.Add("Vote", {
    description = "Cast a vote",
    OnRun = function(self, client, option)
        local steamID = client:SteamID()
        local cooldown = PLUGIN.commandCooldowns[steamID] or 0

        if CurTime() < cooldown then
            return "You must wait before voting again"
        end

        -- Process vote
        PLUGIN.commandCooldowns[steamID] = CurTime() + 60  -- 60 second cooldown

        return "Vote cast!"
    end
})
```

### Context-Sensitive Commands

```lua
ix.command.Add("Unlock", {
    description = "Unlock the door you're looking at",
    OnRun = function(self, client)
        local trace = client:GetEyeTrace()
        local door = trace.Entity

        if not IsValid(door) or not door:IsDoor() then
            return "You must be looking at a door"
        end

        if door:GetPos():Distance(client:GetPos()) > 100 then
            return "Door is too far away"
        end

        door:Fire("unlock")
        return "Door unlocked"
    end
})
```

### Multi-Step Commands

```lua
PLUGIN.confirmations = PLUGIN.confirmations or {}

ix.command.Add("DeleteAll", {
    description = "Delete all items (requires confirmation)",
    adminOnly = true,
    OnRun = function(self, client)
        local steamID = client:SteamID()

        if PLUGIN.confirmations[steamID] then
            -- Already confirmed, execute
            PLUGIN.confirmations[steamID] = nil

            for _, ent in ipairs(ents.FindByClass("ix_item")) do
                ent:Remove()
            end

            return "All items deleted"
        else
            -- Need confirmation
            PLUGIN.confirmations[steamID] = true

            timer.Simple(10, function()
                PLUGIN.confirmations[steamID] = nil
            end)

            return "Type /deleteall again to confirm (10 seconds)"
        end
    end
})
```

## Client-Side Commands

### Creating Client Commands

```lua
-- CLIENT realm only
ix.command.Add("OpenMenu", {
    description = "Opens the custom menu",
    OnRun = function(self, client)
        -- client is LocalPlayer() on client
        vgui.Create("MyCustomMenu")
    end
})
```

### Hybrid Commands

```lua
ix.command.Add("Hybrid", {
    description = "Works on both client and server",
    OnRun = function(self, client)
        if SERVER then
            -- Server-side logic
            return "Server executed"
        else
            -- Client-side logic
            print("Client executed")
        end
    end
})
```

## Command Help

### Automatic Help

The framework automatically generates help text:

```lua
-- /help command
-- Shows all available commands and descriptions

-- /help commandname
-- Shows detailed info about specific command
```

### Custom Help Text

```lua
ix.command.Add("Complex", {
    description = "A complex command",
    syntax = "<player> <amount> [reason]",  -- Custom usage syntax
    arguments = {
        ix.type.player,
        ix.type.number,
        bit.bor(ix.type.text, ix.type.optional)
    },
    OnRun = function(self, client, target, amount, reason)
        -- Command logic
    end
})

-- /help complex
-- Shows: "A complex command"
-- Syntax: /complex <player> <amount> [reason]
```

## Best Practices

### Validation

```lua
ix.command.Add("GiveMoney", {
    description = "Give money to a player",
    adminOnly = true,
    arguments = {ix.type.player, ix.type.number},
    OnRun = function(self, client, target, amount)
        -- Validate amount
        if amount <= 0 then
            return "Amount must be positive"
        end

        if amount > 1000000 then
            return "Amount too large"
        end

        -- Validate target has character
        local char = target:GetCharacter()
        if not char then
            return target:Name() .. " has no active character"
        end

        -- Execute command
        char:GiveMoney(amount)
        return "Gave $" .. amount .. " to " .. target:Name()
    end
})
```

### Error Handling

```lua
ix.command.Add("SafeCommand", {
    description = "Command with error handling",
    OnRun = function(self, client)
        local success, err = pcall(function()
            -- Risky code
            local char = client:GetCharacter()
            local inv = char:GetInventory()
            inv:Add("example_item")
        end)

        if not success then
            ErrorNoHalt("Command error: " .. tostring(err) .. "\n")
            return "An error occurred"
        end

        return "Command successful"
    end
})
```

### User Feedback

```lua
ix.command.Add("Process", {
    description = "Process something",
    OnRun = function(self, client)
        -- Immediate feedback
        client:ChatPrint("Processing...")

        timer.Simple(2, function()
            if IsValid(client) then
                -- Delayed feedback
                client:ChatPrint("Processing complete!")
            end
        end)

        -- Initial return
        return "Started processing"
    end
})
```

### Permission Layers

```lua
ix.command.Add("BanPlayer", {
    description = "Ban a player permanently",
    adminOnly = true,  -- First layer: Must be admin
    OnCheckAccess = function(self, client)
        -- Second layer: Must be super admin OR have ban privilege
        return client:IsSuperAdmin() or CAMI.PlayerHasAccess(client, "Ban")
    end,
    arguments = {ix.type.player, ix.type.text},
    OnRun = function(self, client, target, reason)
        -- Third layer: Can't ban yourself
        if client == target then
            return "You cannot ban yourself"
        end

        -- Fourth layer: Can't ban higher ranked admins
        if target:IsUserGroup("superadmin") and not client:IsSuperAdmin() then
            return "You cannot ban super admins"
        end

        -- All checks passed, execute ban
        target:Ban(0, reason)
        return "Banned " .. target:Name() .. ": " .. reason
    end
})
```

## Debugging Commands

### Debug Output

```lua
ix.command.Add("Debug", {
    description = "Debug command",
    adminOnly = true,
    OnRun = function(self, client)
        print("=== Command Debug ===")
        print("Command:", self.uniqueID)
        print("Player:", client:Name())
        print("SteamID:", client:SteamID())
        print("Position:", client:GetPos())
        print("Character:", client:GetCharacter() and client:GetCharacter():GetName() or "None")
        print("===================")

        return "Debug info printed to console"
    end
})
```

### Testing Commands

```lua
if SERVER then
    concommand.Add("test_command", function(ply, cmd, args)
        if not ply:IsAdmin() then return end

        local target = player.GetByID(1)
        local cmdObj = ix.command.list["givemoney"]

        if cmdObj and IsValid(target) then
            cmdObj:OnRun(ply, target, 100)
        end
    end)
end
```

## Common Patterns

### Give Item Command

```lua
ix.command.Add("GiveItem", {
    description = "Give an item to a player",
    adminOnly = true,
    arguments = {ix.type.player, ix.type.string},
    OnRun = function(self, client, target, itemID)
        local char = target:GetCharacter()
        if not char then
            return "Target has no character"
        end

        local item = ix.item.list[itemID]
        if not item then
            return "Invalid item ID"
        end

        char:GetInventory():Add(itemID)
        return "Gave " .. item.name .. " to " .. target:Name()
    end
})
```

### Set Health Command

```lua
ix.command.Add("SetHP", {
    description = "Set a player's health",
    adminOnly = true,
    arguments = {ix.type.player, ix.type.number},
    OnRun = function(self, client, target, health)
        health = math.Clamp(health, 1, target:GetMaxHealth())
        target:SetHealth(health)
        return "Set " .. target:Name() .. "'s health to " .. health
    end
})
```

### Bring Player Command

```lua
ix.command.Add("Bring", {
    description = "Bring a player to you",
    adminOnly = true,
    arguments = {ix.type.player},
    OnRun = function(self, client, target)
        target:SetPos(client:GetPos() + client:GetForward() * 50)
        return "Brought " .. target:Name()
    end
})
```

## See Also

- [Chat System](chat.md)
- [Configuration System](configuration.md)
- [Flag System](flags.md)
- [Command API Reference](../api/command.md)
