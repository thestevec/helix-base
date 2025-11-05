# Creating Commands for Your Schema

> **Reference**: `gamemode/core/libs/sh_command.lua`, `schema/commands/`

This guide shows you how to create custom chat commands for your schema. Commands allow players to interact with your schema's features through chat.

## ⚠️ Important: Use Helix Command System

**Always use `ix.command.Add()`** rather than creating custom command parsers. The framework provides:
- Automatic argument type validation
- Permission checking (admin, flags, custom)
- Auto-generated help system
- Tab completion for player names
- Alias support
- Consistent command format

## Core Concepts

### Commands in Your Schema

Commands are actions players can perform by typing in chat:

```
/roll 100          - Roll a die
/me waves hello    - Roleplay action
/give player item  - Give items
/broadcast message - Announce to server
```

Command prefixes:
- `/command` - Standard command
- `!command` - Alternative prefix (same as /)
- `@command` - Silent command (no chat output)

### File Location

Create commands in:
```
schema/commands/sh_commandname.lua
```

Or in your plugin:
```
plugins/myplugin/commands/sh_commandname.lua
```

## Creating Your First Command

### Simple Command

```lua
-- File: schema/commands/sh_roll.lua
ix.command.Add("Roll", {
    description = "Roll a die",
    arguments = {
        bit.bor(ix.type.number, ix.type.optional)  -- Optional max number
    },
    OnRun = function(self, client, maxNumber)
        maxNumber = maxNumber or 100

        if SERVER then
            local result = math.random(1, maxNumber)
            local name = client:GetCharacter():GetName()

            -- Announce to nearby players
            local message = name .. " rolled " .. result .. " out of " .. maxNumber
            ix.chat.Send(client, "roll", message)
        end
    end
})
```

Usage:
```
/roll       - Rolls 1-100
/roll 20    - Rolls 1-20
```

### Command with Arguments

```lua
-- File: schema/commands/sh_give.lua
ix.command.Add("Give", {
    description = "Give money to another player",
    arguments = {
        ix.type.player,  -- Target player
        ix.type.number   -- Amount
    },
    OnRun = function(self, client, target, amount)
        local character = client:GetCharacter()
        local targetChar = target:GetCharacter()

        -- Validation
        if not targetChar then
            return "Target has no active character"
        end

        if amount <= 0 then
            return "Amount must be positive"
        end

        if not character:HasMoney(amount) then
            return "You don't have enough money"
        end

        if client:GetPos():Distance(target:GetPos()) > 150 then
            return "You are too far away"
        end

        -- Transfer money
        if SERVER then
            character:TakeMoney(amount)
            targetChar:GiveMoney(amount)

            client:Notify("You gave $" .. amount .. " to " .. target:Name())
            target:Notify(client:Name() .. " gave you $" .. amount)
        end
    end
})
```

Usage:
```
/give "John Doe" 100
```

## Command Structure

### Required Fields

```lua
ix.command.Add("CommandName", {
    description = "What this command does",  -- REQUIRED
    OnRun = function(self, client, ...args)  -- REQUIRED
        -- Command logic
        return "Feedback message"
    end
})
```

### Optional Fields

```lua
ix.command.Add("CommandName", {
    description = "Description",
    adminOnly = false,           -- Only admins
    superAdminOnly = false,      -- Only superadmins
    privilege = "Permission",    -- CAMI privilege
    arguments = {},              -- Argument types
    alias = {"cmd", "alt"},      -- Alternative names
    syntax = "<arg1> [arg2]",    -- Help text

    OnCheckAccess = function(self, client)
        -- Custom permission check
        return client:IsUserGroup("moderator")
    end,

    OnRun = function(self, client, ...args)
        -- Command logic
        return "Result"
    end
})
```

## Complete Command Examples

### Roleplay Action (/me)

```lua
-- File: schema/commands/sh_me.lua
ix.command.Add("Me", {
    description = "Perform a roleplay action",
    arguments = {ix.type.text},  -- All remaining text
    OnRun = function(self, client, action)
        if SERVER then
            local name = client:GetCharacter():GetName()
            local message = "** " .. name .. " " .. action

            -- Send to nearby players
            ix.chat.Send(client, "me", action)
        end
    end
})
```

### Item Giving Command

```lua
-- File: schema/commands/sh_giveitem.lua
ix.command.Add("GiveItem", {
    description = "Give an item to a player",
    adminOnly = true,
    arguments = {
        ix.type.player,  -- Target player
        ix.type.string,  -- Item unique ID
        bit.bor(ix.type.number, ix.type.optional)  -- Optional quantity
    },
    OnRun = function(self, client, target, itemID, quantity)
        quantity = quantity or 1

        local targetChar = target:GetCharacter()
        if not targetChar then
            return "Target has no character"
        end

        local itemTable = ix.item.list[itemID]
        if not itemTable then
            return "Invalid item ID: " .. itemID
        end

        if SERVER then
            local inventory = targetChar:GetInventory()

            for i = 1, quantity do
                inventory:Add(itemID)
            end

            return "Gave " .. quantity .. "x " .. itemTable.name .. " to " .. target:Name()
        end
    end
})
```

### Faction Management Command

```lua
-- File: schema/commands/sh_charsetfaction.lua
ix.command.Add("CharSetFaction", {
    description = "Change a character's faction",
    adminOnly = true,
    arguments = {
        ix.type.player,  -- Target player
        ix.type.string   -- Faction unique ID
    },
    OnRun = function(self, client, target, factionID)
        local character = target:GetCharacter()

        if not character then
            return "Target has no active character"
        end

        local faction = ix.faction.teams[factionID]
        if not faction then
            return "Invalid faction ID: " .. factionID
        end

        if SERVER then
            character:SetFaction(faction.index)

            -- Update model
            local models = faction:GetModels(target)
            if models and #models > 0 then
                character:SetModel(models[1])
                target:SetModel(models[1])
            end

            target:Notify("Your faction has been changed to " .. faction.name)
            return "Changed " .. target:Name() .. "'s faction to " .. faction.name
        end
    end
})
```

### Broadcast Command

```lua
-- File: schema/commands/sh_broadcast.lua
ix.command.Add("Broadcast", {
    description = "Send a server-wide announcement",
    adminOnly = true,
    arguments = {ix.type.text},
    OnRun = function(self, client, message)
        if SERVER then
            for _, ply in ipairs(player.GetAll()) do
                ply:ChatPrint("[BROADCAST] " .. message)
                ply:EmitSound("buttons/button17.wav")
            end

            ix.log.Add(client, "broadcast", message)
        end
    end
})
```

### Flag Management Command

```lua
-- File: schema/commands/sh_flaggive.lua
ix.command.Add("FlagGive", {
    description = "Give a flag to a character",
    adminOnly = true,
    arguments = {
        ix.type.player,
        ix.type.string  -- Flag(s)
    },
    OnRun = function(self, client, target, flags)
        local character = target:GetCharacter()

        if not character then
            return "Target has no character"
        end

        if SERVER then
            character:GiveFlags(flags)

            target:Notify("You have been given flags: " .. flags)
            return "Gave flags '" .. flags .. "' to " .. target:Name()
        end
    end
})

ix.command.Add("FlagTake", {
    description = "Remove a flag from a character",
    adminOnly = true,
    arguments = {
        ix.type.player,
        ix.type.string
    },
    OnRun = function(self, client, target, flags)
        local character = target:GetCharacter()

        if not character then
            return "Target has no character"
        end

        if SERVER then
            character:TakeFlags(flags)

            target:Notify("Your flags have been removed: " .. flags)
            return "Removed flags '" .. flags .. "' from " .. target:Name()
        end
    end
})
```

## Permission Checking

### Admin Only

```lua
ix.command.Add("AdminCommand", {
    description = "Admin-only command",
    adminOnly = true,  -- Requires client:IsAdmin()
    OnRun = function(self, client)
        return "Admin command executed"
    end
})
```

### Flag-Based Permission

```lua
ix.command.Add("VendorMenu", {
    description = "Open vendor menu",
    OnCheckAccess = function(self, client)
        local character = client:GetCharacter()
        -- Require 'v' flag
        return character and character:HasFlags("v")
    end,
    OnRun = function(self, client)
        -- Open vendor UI
        if SERVER then
            net.Start("OpenVendorMenu")
            net.Send(client)
        end
    end
})
```

### Custom Permission Logic

```lua
ix.command.Add("ModeratorKick", {
    description = "Kick a player (moderator+)",
    OnCheckAccess = function(self, client)
        -- Allow moderators and admins
        return client:IsUserGroup("moderator") or
               client:IsUserGroup("admin") or
               client:IsSuperAdmin()
    end,
    arguments = {
        ix.type.player,
        ix.type.text  -- Reason
    },
    OnRun = function(self, client, target, reason)
        target:Kick(reason)
        return "Kicked " .. target:Name() .. ": " .. reason
    end
})
```

## Argument Types

### Available Types

```lua
ix.type.string      -- Single word
ix.type.text        -- Rest of text (all remaining)
ix.type.number      -- Number (int or decimal)
ix.type.player      -- Player with autocomplete
ix.type.character   -- Character name
ix.type.steamid     -- Steam ID
ix.type.bool        -- Boolean (true/false/1/0/yes/no)

-- Modifiers
ix.type.optional    -- Makes argument optional
```

### Using Argument Types

```lua
ix.command.Add("Teleport", {
    description = "Teleport to a player",
    arguments = {
        ix.type.player  -- Required player argument
    },
    OnRun = function(self, client, target)
        client:SetPos(target:GetPos())
        return "Teleported to " .. target:Name()
    end
})

ix.command.Add("SetHealth", {
    description = "Set your health",
    arguments = {
        ix.type.number  -- Required number argument
    },
    OnRun = function(self, client, health)
        health = math.Clamp(health, 1, client:GetMaxHealth())
        client:SetHealth(health)
        return "Health set to " .. health
    end
})

ix.command.Add("Note", {
    description = "Leave a note",
    arguments = {
        ix.type.text  -- Captures all remaining text
    },
    OnRun = function(self, client, noteText)
        client:GetCharacter():SetData("lastNote", noteText)
        return "Note saved: " .. noteText
    end
})
```

### Optional Arguments

```lua
ix.command.Add("Pay", {
    description = "Pay money to a player or yourself",
    arguments = {
        bit.bor(ix.type.player, ix.type.optional),  -- Optional target
        ix.type.number  -- Amount
    },
    OnRun = function(self, client, target, amount)
        -- If no target specified, defaults to self
        target = target or client

        local character = client:GetCharacter()
        local targetChar = target:GetCharacter()

        if not character:HasMoney(amount) then
            return "Not enough money"
        end

        character:TakeMoney(amount)
        targetChar:GiveMoney(amount)

        return "Paid $" .. amount .. " to " .. target:Name()
    end
})
```

## Advanced Features

### Command Aliases

```lua
ix.command.Add("Teleport", {
    description = "Teleport to a player",
    alias = {"tp", "goto", "bring"},  -- Alternative names
    arguments = {ix.type.player},
    OnRun = function(self, client, target)
        client:SetPos(target:GetPos() + Vector(0, 0, 10))
        return "Teleported to " .. target:Name()
    end
})

-- All these work:
-- /teleport "John"
-- /tp "John"
-- /goto "John"
-- /bring "John"
```

### Context-Aware Commands

```lua
ix.command.Add("Unlock", {
    description = "Unlock the door you're looking at",
    OnRun = function(self, client)
        local trace = client:GetEyeTrace()
        local door = trace.Entity

        if not IsValid(door) or not door:IsDoor() then
            return "You must look at a door"
        end

        if trace.HitPos:Distance(client:GetPos()) > 100 then
            return "Door is too far away"
        end

        if SERVER then
            door:Fire("unlock")
            door:EmitSound("doors/door_latch3.wav")
        end

        return "Door unlocked"
    end
})
```

### Cooldown System

```lua
-- In schema/sv_schema.lua
Schema.commandCooldowns = Schema.commandCooldowns or {}

ix.command.Add("Heal", {
    description = "Heal yourself (5 minute cooldown)",
    OnRun = function(self, client)
        local steamID = client:SteamID()
        local cooldown = Schema.commandCooldowns[steamID] or 0

        if CurTime() < cooldown then
            local remaining = math.ceil(cooldown - CurTime())
            return "You must wait " .. remaining .. " seconds"
        end

        if SERVER then
            client:SetHealth(client:GetMaxHealth())
            Schema.commandCooldowns[steamID] = CurTime() + 300  -- 5 minutes

            timer.Simple(300, function()
                if IsValid(client) then
                    client:Notify("Heal command is ready")
                end
            end)
        end

        return "You have been healed"
    end
})
```

### Confirmation Prompts

```lua
ix.command.Add("ClearInventory", {
    description = "Clear your entire inventory (DESTRUCTIVE)",
    OnRun = function(self, client)
        local steamID = client:SteamID()
        Schema.confirmations = Schema.confirmations or {}

        if Schema.confirmations[steamID] then
            -- Confirmed, do action
            Schema.confirmations[steamID] = nil

            if SERVER then
                local inventory = client:GetCharacter():GetInventory()
                for _, item in pairs(inventory:GetItems()) do
                    item:Remove()
                end
            end

            return "Inventory cleared"
        else
            -- Need confirmation
            Schema.confirmations[steamID] = true

            timer.Simple(10, function()
                Schema.confirmations[steamID] = nil
            end)

            return "Type /clearinventory again to confirm (10 seconds)"
        end
    end
})
```

## Best Practices

### ✅ DO

- Create commands in `schema/commands/` or `plugins/*/commands/`
- Use `sh_` prefix for automatic loading
- Always set description
- Validate all arguments on SERVER
- Use appropriate argument types
- Return meaningful feedback messages
- Check permissions properly
- Use OnCheckAccess for custom permissions
- Test commands with different argument combinations
- Provide helpful error messages

### ❌ DON'T

- Don't bypass the command system
- Don't trust client data without validation
- Don't forget SERVER checks for important logic
- Don't create commands without descriptions
- Don't forget to return a value in OnRun
- Don't use wrong argument types
- Don't allow exploits through poor validation
- Don't forget distance checks for interaction commands

## Common Patterns

### Nearby Player Effect

```lua
ix.command.Add("AoEHeal", {
    description = "Heal nearby players",
    OnRun = function(self, client)
        if SERVER then
            local count = 0
            local pos = client:GetPos()

            for _, ply in ipairs(player.GetAll()) do
                if ply:GetPos():Distance(pos) <= 300 then
                    ply:SetHealth(ply:GetMaxHealth())
                    count = count + 1
                end
            end

            return "Healed " .. count .. " nearby players"
        end
    end
})
```

### Data Modification

```lua
ix.command.Add("SetRank", {
    description = "Set your character's rank",
    arguments = {ix.type.number},
    OnRun = function(self, client, rank)
        rank = math.Clamp(rank, 1, 10)

        if SERVER then
            client:GetCharacter():SetData("rank", rank)
        end

        return "Rank set to " .. rank
    end
})
```

### Entity Spawning

```lua
ix.command.Add("SpawnProp", {
    description = "Spawn a prop",
    adminOnly = true,
    arguments = {ix.type.string},  -- Model path
    OnRun = function(self, client, model)
        if SERVER then
            local trace = client:GetEyeTrace()

            local prop = ents.Create("prop_physics")
            prop:SetModel(model)
            prop:SetPos(trace.HitPos + Vector(0, 0, 10))
            prop:Spawn()

            return "Spawned " .. model
        end
    end
})
```

## Testing Commands

1. **Basic Test**: Type command with valid arguments
2. **Invalid Arguments**: Try wrong types/values
3. **Permission Test**: Test with different user groups
4. **Edge Cases**: Empty strings, negative numbers, nil values
5. **Distance Check**: Test range limitations
6. **Multiple Players**: Test interactions between players

## See Also

- [Command System](../systems/commands.md) - Core command system reference
- [Chat System](../systems/chat.md) - Chat integration
- [Flag System](../systems/flags.md) - Permission flags
- [Schema Structure](structure.md) - Schema directory layout
- Source: `gamemode/core/libs/sh_command.lua`
