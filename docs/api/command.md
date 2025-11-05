# Command API (ix.command)

> **Reference**: `gamemode/core/libs/sh_command.lua`

The command API provides registration, parsing, and execution of player commands. Commands can be run through chat with `/CommandName` or via console with `ix CommandName`. The framework includes built-in argument validation, permission checks, and auto-completion.

## ⚠️ Important: Use Built-in Helix Command System

**Always use Helix's built-in command system** rather than creating custom chat hooks. The framework automatically provides:
- Slash command parsing (`/command args`)
- Console command integration (`ix command args`)
- Typed argument validation and conversion
- Optional arguments support
- Permission checks via CAMI
- Automatic syntax generation
- Auto-completion support
- Command cooldown protection
- Logging integration

## Core Concepts

### What are Commands?

Commands are actions players can execute either through chat (`/roll 100`) or console (`ix roll 100`). They:
- Are registered with `ix.command.Add`
- Can have typed arguments with automatic validation
- Support admin/superadmin restrictions
- Have automatic CAMI privilege integration
- Can have aliases for convenience
- Display help text and syntax automatically

### Key Terms

**Command**: An executable action (e.g., "CharGive", "Roll")
**Arguments**: Typed parameters passed to the command
**Syntax**: Auto-generated help text showing required/optional arguments
**Privilege**: CAMI permission required to run the command
**Alias**: Alternative name for a command

## Library Tables

### ix.command.list

**Reference**: `gamemode/core/libs/sh_command.lua:76`

**Realm**: Shared

Table of all registered commands indexed by lowercase name.

```lua
-- Access a command
local rollCmd = ix.command.list["roll"]
print("Description:", rollCmd:GetDescription())

-- List all commands
for name, cmd in pairs(ix.command.list) do
    print(name, cmd.syntax)
end
```

## Library Functions

### ix.command.Add

**Reference**: `gamemode/core/libs/sh_command.lua:157`

**Realm**: Shared

```lua
ix.command.Add(command, data)
```

Creates a new command with specified properties.

**Parameters**:
- `command` (string) - Command name (recommended UpperCamelCase)
- `data` (table) - Command structure:
  - `OnRun` (function) - Command execution callback
  - `description` (string, optional) - Help text (use "@phraseKey" for localized)
  - `arguments` (table, optional) - Typed arguments array
  - `argumentNames` (table, optional) - Custom names for arguments
  - `syntax` (string, optional) - Manual syntax (auto-generated if using `arguments`)
  - `alias` (string/table, optional) - Alternative command names
  - `adminOnly` (bool, optional) - Require admin access
  - `superAdminOnly` (bool, optional) - Require superadmin access
  - `privilege` (string, optional) - Custom CAMI privilege name
  - `OnCheckAccess` (function, optional) - Custom permission check

**Example - Simple Command**:
```lua
ix.command.Add("Roll", {
    description = "Roll a number between 1 and the specified maximum",
    arguments = ix.type.number,
    OnRun = function(self, client, maximum)
        local result = math.random(1, maximum)
        ix.chat.Send(client, "roll", tostring(result), false, nil, {
            max = maximum
        })
    end
})
```

**Example - Admin Command with Multiple Arguments**:
```lua
ix.command.Add("CharGiveMoney", {
    description = "Give money to a character",
    adminOnly = true,
    arguments = {
        ix.type.character,
        ix.type.number
    },
    OnRun = function(self, client, target, amount)
        local character = target
        character:GiveMoney(amount)

        target:Notify("You received " .. ix.currency.Get(amount))
        return "Gave " .. ix.currency.Get(amount) .. " to " .. character:GetName()
    end
})
```

**Example - Optional Arguments**:
```lua
ix.command.Add("Teleport", {
    description = "Teleport to a player or coordinates",
    adminOnly = true,
    arguments = {
        bit.bor(ix.type.player, ix.type.optional),
        bit.bor(ix.type.number, ix.type.optional),
        bit.bor(ix.type.number, ix.type.optional),
        bit.bor(ix.type.number, ix.type.optional)
    },
    OnRun = function(self, client, target, x, y, z)
        if target then
            -- Teleport to player
            client:SetPos(target:GetPos())
            return "Teleported to " .. target:Name()
        elseif x and y and z then
            -- Teleport to coordinates
            client:SetPos(Vector(x, y, z))
            return "Teleported to coordinates"
        else
            return "Usage: /teleport <player> or /teleport <x> <y> <z>"
        end
    end
})
```

**Example - Custom Permission Check**:
```lua
ix.command.Add("AdminMenu", {
    description = "Open admin menu",
    OnCheckAccess = function(self, client)
        -- Custom permission logic
        local char = client:GetCharacter()
        return char and char:HasFlags("A")
    end,
    OnRun = function(self, client)
        net.Start("OpenAdminMenu")
        net.Send(client)
    end
})
```

### ix.command.HasAccess

**Reference**: `gamemode/core/libs/sh_command.lua:313`

**Realm**: Shared

```lua
local hasAccess = ix.command.HasAccess(client, command)
```

Checks if a player can run a specific command.

**Parameters**:
- `client` (Player) - Player to check
- `command` (string) - Command name

**Returns**: (bool) Whether player has access

**Example**:
```lua
-- Check before running
if not ix.command.HasAccess(client, "charban") then
    client:Notify("You don't have permission")
    return
end

-- Show available commands
for name, cmd in pairs(ix.command.list) do
    if ix.command.HasAccess(LocalPlayer(), name) then
        print("Available:", name)
    end
end
```

### ix.command.ExtractArgs

**Reference**: `gamemode/core/libs/sh_command.lua:337`

**Realm**: Shared

```lua
local arguments = ix.command.ExtractArgs(text)
```

Extracts arguments from a string, respecting quoted multi-word arguments.

**Parameters**:
- `text` (string) - String to parse

**Returns**: (table) Array of arguments

**Example**:
```lua
local args = ix.command.ExtractArgs('these are "some arguments"')
-- Result: {"these", "are", "some arguments"}

local args = ix.command.ExtractArgs('/charsetname John "Doe Smith"')
-- Result: {"John", "Doe Smith"}
```

### ix.command.FindAll

**Reference**: `gamemode/core/libs/sh_command.lua:385`

**Realm**: Shared

```lua
local commands = ix.command.FindAll(identifier, bSorted, bReorganize, bRemoveDupes)
```

Finds commands matching a search query.

**Parameters**:
- `identifier` (string) - Search query
- `bSorted` (bool, optional) - Sort results by name
- `bReorganize` (bool, optional) - Move exact matches to top
- `bRemoveDupes` (bool, optional) - Remove commands with duplicate names

**Returns**: (table) Array of matching command tables

**Example**:
```lua
-- Find commands starting with "char"
local cmds = ix.command.FindAll("char", true)
for _, cmd in ipairs(cmds) do
    print(cmd.name)
end

-- Get all commands
local allCmds = ix.command.FindAll("/", true)

-- Search with reorganization
local results = ix.command.FindAll("roll", false, true)
-- Exact match "roll" will be first
```

### ix.command.Run

**Reference**: `gamemode/core/libs/sh_command.lua:462`

**Realm**: Server

```lua
ix.command.Run(client, command, arguments)
```

Forces a player to execute a command programmatically.

**Parameters**:
- `client` (Player) - Player executing the command
- `command` (string) - Command name
- `arguments` (table) - Array of arguments

**Example**:
```lua
-- Make player roll
ix.command.Run(client, "roll", {100})

-- Admin forcing a command
ix.command.Run(targetPlayer, "charfall", {})

-- Execute with multiple arguments
ix.command.Run(client, "chargiveitem", {"weapon_pistol", "1"})
```

**Note**: Still checks permissions and cooldowns.

### ix.command.Parse

**Reference**: `gamemode/core/libs/sh_command.lua:537`

**Realm**: Server

```lua
local found = ix.command.Parse(client, text, realCommand, arguments)
```

Parses a chat string for commands and executes if found.

**Parameters**:
- `client` (Player) - Player executing
- `text` (string) - Input string to parse
- `realCommand` (string, optional) - Specific command to check for
- `arguments` (table, optional) - Pre-extracted arguments

**Returns**: (bool) Whether a command was found

**Example**:
```lua
-- Parse from chat
local isCommand = ix.command.Parse(client, "/roll 100")

-- Parse specific command
ix.command.Parse(client, "100", "roll", {"100"})
```

**Note**: Called automatically by framework from chat hooks.

### ix.command.Send

**Reference**: `gamemode/core/libs/sh_command.lua:607`

**Realm**: Client

```lua
ix.command.Send(command, ...)
```

Requests server to run a command from client.

**Parameters**:
- `command` (string) - Command unique ID
- `...` - Arguments to pass

**Example**:
```lua
-- Request command execution
ix.command.Send("roll", 100)

-- Send command with multiple args
ix.command.Send("chargiveitem", "weapon_pistol", 1)
```

## Argument Types

### Available ix.type Values

**Reference**: `gamemode/core/libs/sh_command.lua:84-145`

```lua
ix.type.string     -- Any string
ix.type.number     -- Numeric value
ix.type.player     -- Valid player entity
ix.type.character  -- Valid character
ix.type.text       -- Rest of input as single string
ix.type.steamid    -- Valid Steam ID
ix.type.bool       -- Boolean value
ix.type.optional   -- Bitwise OR with type to make optional
```

**Usage**:
```lua
arguments = {
    ix.type.character,                         -- Required character
    bit.bor(ix.type.number, ix.type.optional) -- Optional number
}
```

## Complete Examples

### Item Give Command

```lua
ix.command.Add("CharGiveItem", {
    description = "Give an item to a character",
    adminOnly = true,
    arguments = {
        ix.type.character,  -- Target character
        ix.type.string,     -- Item unique ID
        bit.bor(ix.type.number, ix.type.optional)  -- Amount (optional)
    },
    OnRun = function(self, client, target, itemID, amount)
        amount = amount or 1
        local character = target

        -- Give items
        for i = 1, amount do
            if not character:GetInventory():Add(itemID) then
                return "Failed to give item (inventory full?)"
            end
        end

        target:Notify("You received " .. amount .. "x " .. itemID)
        return "Gave " .. amount .. "x " .. itemID .. " to " .. character:GetName()
    end
})
```

### Kick Command with Reason

```lua
ix.command.Add("CharKick", {
    description = "Kick a character back to selection",
    adminOnly = true,
    arguments = {
        ix.type.character,
        ix.type.text  -- Reason (rest of arguments)
    },
    OnRun = function(self, client, target, reason)
        reason = reason or "No reason specified"

        target:Notify("You were kicked: " .. reason)
        target:GetCharacter():Kick()

        ix.util.Notify(reason, target)
        return "Kicked " .. target:Name() .. " for: " .. reason
    end
})
```

### Command with Alias

```lua
ix.command.Add("PrivateMessage", {
    description = "Send a private message",
    alias = {"pm", "msg", "whisper"},
    arguments = {
        ix.type.player,
        ix.type.text
    },
    OnRun = function(self, client, target, message)
        target:ChatPrint("[PM from " .. client:Name() .. "]: " .. message)
        client:ChatPrint("[PM to " .. target:Name() .. "]: " .. message)
    end
})
-- Now usable as: /pm, /msg, /whisper, or /privatemessage
```

### Command with Custom Validation

```lua
ix.command.Add("SetHealth", {
    description = "Set a player's health",
    adminOnly = true,
    arguments = {
        ix.type.player,
        ix.type.number
    },
    OnRun = function(self, client, target, amount)
        -- Validate range
        if amount < 1 or amount > 1000 then
            return "Health must be between 1 and 1000"
        end

        target:SetHealth(amount)
        return "Set " .. target:Name() .. "'s health to " .. amount
    end
})
```

## Best Practices

### ✅ DO

- Use UpperCamelCase for command names
- Specify argument types for automatic validation
- Use `ix.type.text` for the last argument if it should capture remaining input
- Return strings from `OnRun` to notify the player
- Use `adminOnly` or `superAdminOnly` for restricted commands
- Provide clear, helpful descriptions
- Use localized phrases with "@phraseKey" format
- Add aliases for commonly used commands

### ❌ DON'T

- Don't create commands without `OnRun` callbacks
- Don't use `ix.type.optional` without bitwise OR
- Don't put `ix.type.text` before the last argument
- Don't forget realm restrictions (Run/Parse are server-only)
- Don't bypass permission checks
- Don't create duplicate command names
- Don't use spaces in command names
- Don't forget to validate custom argument ranges

## Common Patterns

### Return Values for Feedback

```lua
OnRun = function(self, client, target)
    if not IsValid(target) then
        return "Invalid target"  -- Shows error to player
    end

    -- Success
    return "Command executed successfully"

    -- Localized feedback
    return "@commandSuccess"  -- Uses language phrase
end
```

### Admin Command with Logging

```lua
ix.command.Add("CharSetMoney", {
    description = "Set a character's money",
    adminOnly = true,
    arguments = {ix.type.character, ix.type.number},
    OnRun = function(self, client, target, amount)
        local character = target
        local oldMoney = character:GetMoney()

        character:SetMoney(amount)

        -- Log the action
        ix.log.Add(client, "moneySet", target:Name(), oldMoney, amount)

        return string.format("Set %s's money from %s to %s",
            character:GetName(),
            ix.currency.Get(oldMoney),
            ix.currency.Get(amount)
        )
    end
})
```

### Character-Only Command

```lua
ix.command.Add("MyCommand", {
    description = "Command that requires a character",
    OnRun = function(self, client)
        local character = client:GetCharacter()

        if not character then
            return "You need an active character"
        end

        -- Do something with character
        character:DoSomething()
    end
})
```

## Common Issues

### Arguments Not Validating

**Cause**: Wrong argument type or missing optional flag.

**Fix**: Use correct types:
```lua
-- WRONG
arguments = {ix.type.optional, ix.type.number}  -- Can't use optional alone

-- CORRECT
arguments = {bit.bor(ix.type.number, ix.type.optional)}
```

### "Text argument outside of last argument"

**Cause**: Using `ix.type.text` before the last argument.

**Fix**: Only use text as final argument:
```lua
-- WRONG
arguments = {ix.type.text, ix.type.number}

-- CORRECT
arguments = {ix.type.number, ix.type.text}
```

### Command Not Showing in Auto-complete

**Cause**: `OnCheckAccess` returning false or not having permission.

**Fix**: Check permissions:
```lua
-- Debug access
OnCheckAccess = function(self, client)
    print("Checking access for", client:Name())
    return true  -- Temporarily allow all
end
```

### "Invalid argument #N"

**Cause**: Player provided wrong argument type.

**Solution**: Arguments are auto-validated. This error tells the player they used wrong input.

## Related Hooks

Commands don't have specific hooks but integrate with:

### PlayerSay

Used internally to detect chat commands.

```lua
hook.Add("PlayerSay", "MyCommandHook", function(client, text)
    -- Framework handles this automatically
    -- Don't use PlayerSay for commands, use ix.command.Add
end)
```

## See Also

- [Chat API](chat.md) - Chat system that commands integrate with
- [Plugin API](plugin.md) - Creating plugins that add commands
- [Character API](character.md) - Character operations in commands
- [Flag API](flag.md) - Permission flags for command access
- Source: `gamemode/core/libs/sh_command.lua`
