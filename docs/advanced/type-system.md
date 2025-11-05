# Type System (ix.type)

> **Reference**: `gamemode/core/sh_util.lua:5`, `gamemode/core/libs/sh_command.lua`

The type system provides enumerated constants for defining command arguments, enabling automatic validation and parsing of user input with type safety.

## ⚠️ Important: Use Built-in Type System

**Always use Helix's type constants** for command arguments rather than manual string parsing. The framework provides:
- Automatic argument validation and parsing
- Type-safe command definitions
- Player/character lookup with error handling
- Optional argument support
- Array argument support
- Automatic syntax generation
- Consistent error messages

## Core Concepts

### What is the Type System?

The type system is a set of bitwise constants used primarily in command argument definitions. When you specify argument types, Helix automatically validates and parses user input, eliminating boilerplate validation code.

**Benefits**:
- No manual string parsing
- Automatic type conversion
- Built-in error messages
- Player/character name completion
- Optional and array arguments
- Type safety at runtime

### Key Terms

- **Type Constant**: A bitwise flag representing an argument type (e.g., `ix.type.string`)
- **Optional Type**: Type modifier using bitwise OR to make argument optional
- **Array Type**: Type modifier for accepting multiple values
- **Type Validation**: Automatic checking that user input matches expected type
- **Type Parsing**: Converting string input to proper Lua type

## Available Types

**Reference**: `gamemode/core/sh_util.lua:5-28`

### Basic Types

```lua
ix.type.string      -- Single word (no spaces)
ix.type.text        -- Rest of text (all remaining words)
ix.type.number      -- Any number (integer or decimal)
ix.type.bool        -- Boolean (true/false, 1/0, yes/no)
```

### Special Types

```lua
ix.type.player      -- Player name (with tab completion)
ix.type.character   -- Character name (finds player, then character)
ix.type.steamid     -- Steam ID (STEAM_X:Y:Z format)
ix.type.color       -- Color (r,g,b or r,g,b,a)
ix.type.vector      -- Vector (x,y,z)
```

### Modifiers

```lua
ix.type.optional    -- Makes argument optional (use with bit.bor)
ix.type.array       -- Accepts multiple values (use with bit.bor)
```

## Using Types in Commands

### Basic Type Usage

**Reference**: `gamemode/core/libs/sh_command.lua:80-149`

```lua
COMMAND.arguments = {
    ix.type.player,    -- First argument: player
    ix.type.number     -- Second argument: number
}

function COMMAND:OnRun(client, target, amount)
    -- target is validated Player entity
    -- amount is validated number
end
```

**Complete Example**:
```lua
-- Give item command
ix.command.Add("GiveItem", {
    description = "Give an item to a player",
    adminOnly = true,
    arguments = {
        ix.type.player,    -- Target player
        ix.type.string     -- Item unique ID
    },
    OnRun = function(self, client, target, itemID)
        -- Both arguments guaranteed valid
        local character = target:GetCharacter()

        if not character then
            return "Player doesn't have a character"
        end

        character:GetInventory():Add(itemID, 1, nil, nil, function(item)
            if item then
                client:Notify("Gave " .. item:GetName() .. " to " .. target:Name())
                target:Notify("You received " .. item:GetName())
            else
                client:Notify("Failed to give item")
            end
        end)
    end
})

-- Set attribute command
ix.command.Add("SetAttribute", {
    description = "Set a character's attribute",
    superAdminOnly = true,
    arguments = {
        ix.type.character,  -- Target character
        ix.type.string,     -- Attribute name
        ix.type.number      -- New value
    },
    OnRun = function(self, client, target, attributeName, value)
        -- target is Character object (not Player!)
        target:SetAttribute(attributeName, value)

        local player = target:GetPlayer()
        if IsValid(player) then
            player:Notify("Your " .. attributeName .. " was set to " .. value)
        end

        client:Notify("Set " .. target:GetName() .. "'s " .. attributeName .. " to " .. value)
    end
})

-- Teleport command
ix.command.Add("Teleport", {
    description = "Teleport to coordinates",
    adminOnly = true,
    arguments = {
        ix.type.vector  -- Position vector
    },
    OnRun = function(self, client, position)
        -- position is validated Vector
        client:SetPos(position)
        client:Notify("Teleported to " .. tostring(position))
    end
})
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't manually parse arguments
function COMMAND:OnRun(client, arguments)
    local targetName = arguments[1]
    local target = nil

    for _, v in ipairs(player.GetAll()) do
        if v:Name() == targetName then
            target = v
            break
        end
    end

    if not target then
        return "Player not found"  -- Use ix.type.player instead!
    end
end

-- WRONG: Don't skip type definitions
COMMAND.arguments = nil  -- Define proper types!

function COMMAND:OnRun(client, arguments)
    local amount = tonumber(arguments[1])  -- Use ix.type.number!
    if not amount then
        return "Invalid number"
    end
end
```

### Optional Arguments

**Reference**: `gamemode/core/libs/sh_command.lua:49-50`

```lua
arguments = {
    ix.type.player,                                      -- Required
    bit.bor(ix.type.number, ix.type.optional),          -- Optional number
    bit.bor(ix.type.text, ix.type.optional)             -- Optional text
}
```

**Complete Example**:
```lua
-- Kick command with optional reason
ix.command.Add("Kick", {
    description = "Kick a player from the server",
    adminOnly = true,
    arguments = {
        ix.type.player,                              -- Required: target
        bit.bor(ix.type.text, ix.type.optional)     -- Optional: reason
    },
    OnRun = function(self, client, target, reason)
        reason = reason or "No reason provided"  -- Default if nil

        target:Kick(reason)
        ix.log.Add(client, "playerKick", target:Name(), reason)
    end
})

-- Teleport to player or position
ix.command.Add("Goto", {
    description = "Teleport to player or position",
    adminOnly = true,
    arguments = {
        bit.bor(ix.type.player, ix.type.optional),   -- Optional: target player
        bit.bor(ix.type.vector, ix.type.optional)    -- Optional: position
    },
    OnRun = function(self, client, target, position)
        if target then
            -- Teleport to player
            client:SetPos(target:GetPos())
            client:Notify("Teleported to " .. target:Name())
        elseif position then
            -- Teleport to position
            client:SetPos(position)
            client:Notify("Teleported to " .. tostring(position))
        else
            -- Teleport to look position
            local trace = client:GetEyeTrace()
            client:SetPos(trace.HitPos)
            client:Notify("Teleported to look position")
        end
    end
})

-- Give money with optional target
ix.command.Add("GiveMoney", {
    description = "Give money to yourself or another player",
    adminOnly = true,
    arguments = {
        ix.type.number,                              -- Required: amount
        bit.bor(ix.type.player, ix.type.optional)   -- Optional: target
    },
    OnRun = function(self, client, amount, target)
        target = target or client  -- Default to self if no target

        local character = target:GetCharacter()

        if not character then
            return "Target doesn't have a character"
        end

        character:GiveMoney(amount)
        target:Notify("You received " .. ix.currency.Get(amount))

        if target != client then
            client:Notify("Gave " .. ix.currency.Get(amount) .. " to " .. target:Name())
        end
    end
})
```

### Text Type (Multiple Words)

**Reference**: `gamemode/core/libs/sh_command.lua:99-101`

```lua
arguments = {
    ix.type.player,  -- First argument
    ix.type.text     -- Consumes all remaining text
}
```

**Complete Example**:
```lua
-- Announcement command
ix.command.Add("Announce", {
    description = "Send an announcement to all players",
    adminOnly = true,
    arguments = {
        ix.type.text  -- All remaining text
    },
    OnRun = function(self, client, message)
        -- message contains all text after command
        for _, v in ipairs(player.GetAll()) do
            v:ChatPrint("[ANNOUNCEMENT] " .. message)
        end

        ix.log.Add(client, "announcement", message)
    end
})

-- Private message with reason
ix.command.Add("PM", {
    description = "Send a private message",
    arguments = {
        ix.type.player,  -- Target
        ix.type.text     -- Message (rest of text)
    },
    OnRun = function(self, client, target, message)
        target:ChatPrint("[PM from " .. client:Name() .. "] " .. message)
        client:ChatPrint("[PM to " .. target:Name() .. "] " .. message)
    end
})

-- Set character description
ix.command.Add("CharSetDesc", {
    description = "Set a character's description",
    adminOnly = true,
    arguments = {
        ix.type.character,  -- Target character
        ix.type.text        -- New description
    },
    OnRun = function(self, client, character, description)
        character:SetDescription(description)
        character:Save()

        client:Notify("Set description for " .. character:GetName())

        local player = character:GetPlayer()
        if IsValid(player) then
            player:Notify("Your description was updated")
        end
    end
})
```

### Array Arguments (Multiple Values)

**Reference**: `gamemode/core/libs/sh_command.lua:26`

```lua
arguments = {
    bit.bor(ix.type.player, ix.type.array),  -- Multiple players
    ix.type.string                            -- Single string
}
```

**Complete Example**:
```lua
-- Give item to multiple players
ix.command.Add("GiveItemMulti", {
    description = "Give an item to multiple players",
    adminOnly = true,
    arguments = {
        bit.bor(ix.type.player, ix.type.array),  -- Multiple targets
        ix.type.string                            -- Item ID
    },
    OnRun = function(self, client, targets, itemID)
        -- targets is an array of Player entities
        local count = 0

        for _, target in ipairs(targets) do
            local character = target:GetCharacter()

            if character then
                character:GetInventory():Add(itemID, 1, nil, nil, function(item)
                    if item then
                        target:Notify("You received " .. item:GetName())
                        count = count + 1
                    end
                end)
            end
        end

        client:Notify("Gave item to " .. count .. " players")
    end
})

-- Slap multiple players
ix.command.Add("SlapMulti", {
    description = "Slap multiple players",
    adminOnly = true,
    arguments = {
        bit.bor(ix.type.player, ix.type.array),  -- Multiple targets
        ix.type.number                            -- Damage
    },
    OnRun = function(self, client, targets, damage)
        for _, target in ipairs(targets) do
            target:TakeDamage(damage, client, client)
            target:SetVelocity(Vector(0, 0, 300))
        end

        client:Notify("Slapped " .. #targets .. " players")
    end
})
```

### Type Combinations

```lua
-- Optional array
arguments = {
    bit.bor(ix.type.player, ix.type.array, ix.type.optional)
}

-- Multiple modifiers
arguments = {
    ix.type.character,                                      -- Required character
    bit.bor(ix.type.string, ix.type.optional),             -- Optional string
    bit.bor(ix.type.number, ix.type.array)                 -- Array of numbers
}
```

**Complete Example**:
```lua
-- Flexible teleport command
ix.command.Add("Bring", {
    description = "Teleport player(s) to you",
    adminOnly = true,
    arguments = {
        bit.bor(ix.type.player, ix.type.array, ix.type.optional)
    },
    OnRun = function(self, client, targets)
        if not targets then
            -- No targets specified, bring all players
            targets = player.GetAll()
        end

        local position = client:GetPos()

        for _, target in ipairs(targets) do
            if target != client then
                target:SetPos(position)
            end
        end

        client:Notify("Brought " .. #targets .. " players")
    end
})
```

## Type Validation Details

### String Type

- Accepts single word (no spaces)
- Converts to Lua string
- Cannot be empty

### Text Type

- Accepts all remaining text (multiple words)
- Consumes rest of command line
- Must be last argument
- Can be empty string

### Number Type

- Accepts integers and decimals
- Converts to Lua number
- Returns `nil` if optional and not provided
- Fails validation if not a number

### Player Type

- Searches for player by partial name
- Uses `ix.util.FindPlayer()` internally
- Provides automatic error messages
- Returns Player entity
- Fails if player not found (unless optional)

### Character Type

- Searches for player, then gets character
- Returns Character object (not Player!)
- Fails if player has no character
- Use when you need character, not player

### SteamID Type

- Validates STEAM_X:Y:Z format
- Returns string if valid
- Fails validation if format incorrect

### Boolean Type

- Accepts: `true`/`false`, `1`/`0`, `yes`/`no`
- Converts to Lua boolean
- Case insensitive

### Color Type

- Accepts: `r,g,b` or `r,g,b,a`
- Converts to Color object
- Components must be 0-255

### Vector Type

- Accepts: `x,y,z`
- Converts to Vector object
- Components can be any number

## Complete Example

### Multi-Purpose Admin Command

```lua
ix.command.Add("AdminTool", {
    description = "Swiss army knife admin command",
    superAdminOnly = true,
    arguments = {
        ix.type.string,                                     -- Action type
        bit.bor(ix.type.player, ix.type.array),            -- Target(s)
        bit.bor(ix.type.number, ix.type.optional),         -- Amount/value
        bit.bor(ix.type.text, ix.type.optional)            -- Reason/message
    },
    OnRun = function(self, client, action, targets, value, message)
        action = action:lower()

        if action == "heal" then
            for _, target in ipairs(targets) do
                target:SetHealth(target:GetMaxHealth())
                target:Notify("You were healed")
            end

        elseif action == "money" then
            value = value or 100

            for _, target in ipairs(targets) do
                local character = target:GetCharacter()

                if character then
                    character:GiveMoney(value)
                    target:Notify("You received " .. ix.currency.Get(value))
                end
            end

        elseif action == "notify" then
            message = message or "Admin notification"

            for _, target in ipairs(targets) do
                target:Notify(message)
            end

        else
            return "Invalid action: " .. action
        end

        client:Notify("Executed " .. action .. " on " .. #targets .. " players")
    end
})
```

## Best Practices

### ✅ DO

- Use `ix.type` constants for all command arguments
- Make arguments optional when appropriate with `bit.bor`
- Use `ix.type.text` for multi-word input
- Use `ix.type.character` when you need character data
- Use `ix.type.player` when you need player entity
- Check optional arguments with `if value then`
- Document argument purpose in command description

### ❌ DON'T

- Don't manually parse command arguments
- Don't use `ix.type.text` for non-final arguments
- Don't forget to validate optional arguments before use
- Don't mix up `ix.type.player` and `ix.type.character`
- Don't create custom type validation
- Don't bypass type system with `arguments = nil`
- Don't assume optional arguments are always provided

## Common Patterns

### Pattern 1: Target Self or Other

```lua
arguments = {
    bit.bor(ix.type.player, ix.type.optional)
},
OnRun = function(self, client, target)
    target = target or client  -- Default to self
    -- ...
end
```

### Pattern 2: Required + Optional Text

```lua
arguments = {
    ix.type.player,
    bit.bor(ix.type.text, ix.type.optional)
},
OnRun = function(self, client, target, reason)
    reason = reason or "No reason provided"
    -- ...
end
```

### Pattern 3: Multiple Targets

```lua
arguments = {
    bit.bor(ix.type.player, ix.type.array)
},
OnRun = function(self, client, targets)
    for _, target in ipairs(targets) do
        -- Process each target
    end
end
```

## See Also

- [Commands System](../systems/commands.md) - Complete command system documentation
- [Player Meta](./meta-tables.md) - Player methods
- [Character Meta](./meta-tables.md) - Character methods
- Source: `gamemode/core/sh_util.lua`
- Source: `gamemode/core/libs/sh_command.lua`
