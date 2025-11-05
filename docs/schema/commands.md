# Creating Schema Commands

> **Reference**: `gamemode/core/libs/sh_command.lua`, `schema/commands/`

Commands are chat-based actions players can execute. This guide shows how to create commands specifically for your schema.

## ⚠️ Important: Use Helix Command System

**Always use `ix.command.Add()`** rather than creating custom command parsers. The framework provides:
- Automatic argument type validation
- Permission checking
- Help text generation
- Tab completion
- Command aliasing

## Core Concepts

### What are Schema Commands?

Schema commands are roleplay-specific actions:
- **/request** - Request items from CPs (HL2RP)
- **/radio** - Send message on radio frequency
- **/me** - Perform an action (usually provided by framework)
- **/roll** - Roll dice for chance events
- **/givepermit** - Issue work permits (HL2RP)

Commands enhance roleplay by:
- Providing structure to common actions
- Validating permissions automatically
- Creating consistent interactions

## Creating Commands

### File Location

**Place command files** in:
```
schema/commands/sh_commands.lua
```

Or organize by purpose:
```
schema/commands/
├── sh_commands.lua      -- General commands
├── sh_admin.lua         -- Admin commands
└── sh_roleplay.lua      -- RP commands
```

### Basic Command Template

```lua
-- schema/commands/sh_commands.lua
ix.command.Add("Request", {
    description = "Request supplies from Civil Protection",
    OnRun = function(self, client)
        local character = client:GetCharacter()

        if character:GetFaction() != FACTION_CITIZEN then
            return "Only citizens can request supplies"
        end

        -- Find nearby CP
        local nearbyCP
        for _, ply in ipairs(player.GetAll()) do
            if ply:GetCharacter() and ply:GetCharacter():GetFaction() == FACTION_CP then
                if ply:GetPos():Distance(client:GetPos()) <= 300 then
                    nearbyCP = ply
                    break
                end
            end
        end

        if not nearbyCP then
            return "No Civil Protection nearby"
        end

        -- Notify CP
        nearbyCP:ChatPrint(client:Name() .. " is requesting supplies")
        client:ChatPrint("You requested supplies from " .. nearbyCP:Name())
    end
})
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't create custom chat parsers
hook.Add("PlayerSay", "CustomCommands", function(ply, text)
    if string.sub(text, 1, 1) == "/" then
        -- Don't do this!
    end
end)
```

## Required Command Properties

### Minimal Requirements

```lua
ix.command.Add("CommandName", {
    description = "What the command does",    -- REQUIRED
    OnRun = function(self, client)            -- REQUIRED
        -- Command logic
        return "Result message"
    end
})
```

## Common Command Examples

### Request Command (HL2RP)

```lua
ix.command.Add("Request", {
    description = "Request items from nearby Civil Protection",
    arguments = {ix.type.text},  -- Item request text
    OnRun = function(self, client, text)
        local character = client:GetCharacter()

        -- Only citizens can request
        if character:GetFaction() != FACTION_CITIZEN then
            return "Only citizens can make requests"
        end

        -- Check cooldown
        local lastRequest = character:GetData("lastRequest", 0)
        if CurTime() - lastRequest < 60 then
            return "You must wait before making another request"
        end

        -- Find nearby CP
        local nearbyCP
        for _, ply in ipairs(player.GetAll()) do
            local plyChr = ply:GetCharacter()
            if plyChr and plyChr:GetFaction() == FACTION_CP then
                if ply:GetPos():Distance(client:GetPos()) <= 300 then
                    nearbyCP = ply
                    break
                end
            end
        end

        if not nearbyCP then
            return "No Civil Protection nearby"
        end

        -- Set cooldown
        character:SetData("lastRequest", CurTime())

        -- Notify CP and nearby players
        ix.chat.Send(client, "request", text)

        return false  -- Don't show return message
    end
})
```

### Roll Command

```lua
ix.command.Add("Roll", {
    description = "Roll a random number",
    arguments = {
        bit.bor(ix.type.number, ix.type.optional)  -- Optional max number
    },
    OnRun = function(self, client, maxNumber)
        maxNumber = maxNumber or 100

        if maxNumber < 1 or maxNumber > 1000 then
            return "Number must be between 1 and 1000"
        end

        local roll = math.random(1, maxNumber)

        -- Announce to nearby players
        for _, ply in ipairs(player.GetAll()) do
            if ply:GetPos():Distance(client:GetPos()) <= 500 then
                ply:ChatPrint(client:Name() .. " rolls " .. roll .. " out of " .. maxNumber)
            end
        end

        return false  -- Message already sent
    end
})
```

### Radio Command

```lua
ix.command.Add("Radio", {
    description = "Send message on your radio frequency",
    arguments = {ix.type.text},
    OnRun = function(self, client, message)
        local character = client:GetCharacter()
        local inventory = character:GetInventory()

        -- Check if player has radio
        local hasRadio = false
        local radioFreq = "100.0"

        for _, item in pairs(inventory:GetItems()) do
            if item.uniqueID == "handheld_radio" then
                if item:GetData("enabled", false) then
                    hasRadio = true
                    radioFreq = item:GetData("frequency", "100.0")
                    break
                end
            end
        end

        if not hasRadio then
            return "You don't have an active radio"
        end

        -- Send to players on same frequency
        for _, ply in ipairs(player.GetAll()) do
            local plyChr = ply:GetCharacter()
            if not plyChr then continue end

            local plyInv = plyChr:GetInventory()
            for _, item in pairs(plyInv:GetItems()) do
                if item.uniqueID == "handheld_radio" then
                    if item:GetData("enabled", false) and item:GetData("frequency") == radioFreq then
                        ply:ChatPrint("[RADIO " .. radioFreq .. "] " .. client:Name() .. ": " .. message)
                    end
                end
            end
        end

        return false  -- Message sent via chat
    end
})
```

### Give Permit (Admin)

```lua
ix.command.Add("GivePermit", {
    description = "Issue a work permit to a citizen",
    adminOnly = true,
    arguments = {
        ix.type.player,   -- Target player
        ix.type.string    -- Permit type
    },
    OnRun = function(self, client, target, permitType)
        local character = target:GetCharacter()

        if not character then
            return "Target has no character"
        end

        if character:GetFaction() != FACTION_CITIZEN then
            return "Target must be a citizen"
        end

        local validPermits = {
            "work", "travel", "medical", "ration"
        }

        if not table.HasValue(validPermits, permitType) then
            return "Invalid permit type. Valid: " .. table.concat(validPermits, ", ")
        end

        -- Give permit item
        local inventory = character:GetInventory()
        inventory:Add("permit_" .. permitType)

        -- Log action
        ix.log.Add(client, "givePermit", permitType, target)

        return "Gave " .. permitType .. " permit to " .. target:Name()
    end
})
```

### Check Inventory Command

```lua
ix.command.Add("CheckInv", {
    description = "Check a player's inventory",
    adminOnly = true,
    arguments = {ix.type.player},
    OnRun = function(self, client, target)
        local character = target:GetCharacter()

        if not character then
            return "Target has no character"
        end

        local inventory = character:GetInventory()
        local items = inventory:GetItems()

        client:ChatPrint("=== " .. target:Name() .. "'s Inventory ===")

        for _, item in pairs(items) do
            client:ChatPrint("- " .. item.name .. " (x" .. (item:GetData("amount", 1)) .. ")")
        end

        client:ChatPrint("Money: " .. ix.currency.Get(character:GetMoney()))

        return false
    end
})
```

## Command Arguments

### Argument Types

```lua
ix.type.string      -- Single word
ix.type.text        -- Rest of text
ix.type.number      -- Any number
ix.type.player      -- Player name
ix.type.character   -- Character name
ix.type.bool        -- true/false
```

### Optional Arguments

```lua
arguments = {
    ix.type.player,                                -- Required
    bit.bor(ix.type.number, ix.type.optional)      -- Optional
}
```

### Multiple Arguments Example

```lua
ix.command.Add("GiveItem", {
    description = "Give item to player",
    adminOnly = true,
    arguments = {
        ix.type.player,   -- Who to give to
        ix.type.string,   -- Item uniqueID
        ix.type.number    -- Quantity
    },
    OnRun = function(self, client, target, itemID, quantity)
        local character = target:GetCharacter()

        if not character then
            return "Target has no character"
        end

        local inventory = character:GetInventory()

        for i = 1, quantity do
            inventory:Add(itemID)
        end

        return "Gave " .. quantity .. "x " .. itemID .. " to " .. target:Name()
    end
})
```

## Permission Checking

### Admin Only

```lua
ix.command.Add("AdminCommand", {
    description = "Admin only command",
    adminOnly = true,
    OnRun = function(self, client)
        return "Admin command executed"
    end
})
```

### Custom Permission Check

```lua
ix.command.Add("CPCommand", {
    description = "Civil Protection only command",
    OnCheckAccess = function(self, client)
        local character = client:GetCharacter()
        return character and character:GetFaction() == FACTION_CP
    end,
    OnRun = function(self, client)
        return "CP command executed"
    end
})
```

### Flag-Based Permission

```lua
ix.command.Add("VendorCommand", {
    description = "Vendor flag required",
    OnCheckAccess = function(self, client)
        local character = client:GetCharacter()
        return character and character:HasFlags("v")
    end,
    OnRun = function(self, client)
        return "Vendor command executed"
    end
})
```

## Schema-Specific Patterns

### Proximity-Based Commands

```lua
ix.command.Add("Search", {
    description = "Search a nearby player",
    arguments = {ix.type.player},
    OnCheckAccess = function(self, client)
        local character = client:GetCharacter()
        return character and character:GetFaction() == FACTION_CP
    end,
    OnRun = function(self, client, target)
        -- Check distance
        if client:GetPos():Distance(target:GetPos()) > 150 then
            return "Target too far away"
        end

        -- Check if target is restrained
        local targetChar = target:GetCharacter()
        if not targetChar:GetData("tied") then
            return "Target must be restrained first"
        end

        -- Show inventory
        local inventory = targetChar:GetInventory()
        client:ChatPrint("=== Searching " .. target:Name() .. " ===")

        for _, item in pairs(inventory:GetItems()) do
            client:ChatPrint("- " .. item.name)
        end

        -- Notify target
        target:ChatPrint(client:Name() .. " is searching you")

        return false
    end
})
```

### Cooldown System

```lua
PLUGIN.commandCooldowns = PLUGIN.commandCooldowns or {}

ix.command.Add("Broadcast", {
    description = "Broadcast a message (5 min cooldown)",
    arguments = {ix.type.text},
    OnRun = function(self, client, message)
        local steamID = client:SteamID()
        local cooldown = PLUGIN.commandCooldowns[steamID] or 0

        if CurTime() < cooldown then
            local remaining = math.ceil(cooldown - CurTime())
            return "Must wait " .. remaining .. " seconds"
        end

        -- Set cooldown (5 minutes)
        PLUGIN.commandCooldowns[steamID] = CurTime() + 300

        -- Broadcast
        for _, ply in ipairs(player.GetAll()) do
            ply:ChatPrint("[BROADCAST] " .. message)
        end

        return false
    end
})
```

### Context-Sensitive Commands

```lua
ix.command.Add("Lockdown", {
    description = "Initiate city lockdown",
    OnCheckAccess = function(self, client)
        local character = client:GetCharacter()
        if not character then return false end

        -- Must be CP with rank 5+
        if character:GetFaction() != FACTION_CP then
            return false
        end

        if character:GetData("rank", 0) < 5 then
            return false
        end

        return true
    end,
    OnRun = function(self, client)
        -- Check if already in lockdown
        if Schema.lockdownActive then
            return "Lockdown already active"
        end

        Schema.lockdownActive = true

        -- Notify all players
        for _, ply in ipairs(player.GetAll()) do
            ply:ChatPrint("[COMBINE OVERWATCH] City lockdown initiated")
            ply:EmitSound("npc/overwatch/cityvoice/f_lockdownterminated_spkr.wav")
        end

        -- Lock all doors
        for _, door in ipairs(ents.FindByClass("prop_door*")) do
            door:Fire("lock")
        end

        -- Log action
        ix.log.Add(client, "lockdown", "initiated")

        return false
    end
})
```

## Advanced Features

### Multi-Step Commands

```lua
PLUGIN.pendingTransfers = PLUGIN.pendingTransfers or {}

ix.command.Add("Transfer", {
    description = "Transfer character to another player",
    adminOnly = true,
    arguments = {
        ix.type.character,
        ix.type.player
    },
    OnRun = function(self, client, character, target)
        local charID = character:GetID()

        -- First confirmation
        if not PLUGIN.pendingTransfers[charID] then
            PLUGIN.pendingTransfers[charID] = {
                target = target,
                admin = client
            }

            timer.Simple(10, function()
                PLUGIN.pendingTransfers[charID] = nil
            end)

            return "Type command again to confirm transfer of '" .. character:GetName() .. "' to " .. target:Name()
        end

        -- Confirmed
        PLUGIN.pendingTransfers[charID] = nil

        -- Do transfer
        character:SetOwner(target:SteamID64())
        character:Save()

        ix.log.Add(client, "transferChar", character:GetName(), target)

        return "Transferred character to " .. target:Name()
    end
})
```

### Command Aliases

```lua
ix.command.Add("Teleport", {
    description = "Teleport to a player",
    alias = {"tp", "goto"},
    adminOnly = true,
    arguments = {ix.type.player},
    OnRun = function(self, client, target)
        client:SetPos(target:GetPos() + Vector(0, 0, 10))
        return "Teleported to " .. target:Name()
    end
})

-- All work: /teleport, /tp, /goto
```

## Best Practices

### ✅ DO

- Place commands in `schema/commands/` folder
- Use `ix.command.Add()` for all commands
- Set clear, descriptive descriptions
- Validate all input in OnRun
- Check permissions with OnCheckAccess
- Return helpful error messages
- Use appropriate argument types
- Log important admin actions with ix.log.Add()
- Test edge cases (nil values, disconnects)

### ❌ DON'T

- Don't create custom chat command parsers
- Don't forget permission checks
- Don't trust client-provided data
- Don't forget to validate arguments
- Don't skip error messages
- Don't create commands without descriptions
- Don't bypass the command system

## Common Patterns

### Admin Command Template

```lua
ix.command.Add("AdminAction", {
    description = "Perform admin action",
    adminOnly = true,
    arguments = {ix.type.player, ix.type.string},
    OnRun = function(self, client, target, reason)
        -- Validate target
        if not IsValid(target) then
            return "Invalid target"
        end

        -- Perform action
        -- ... do something ...

        -- Log it
        ix.log.Add(client, "adminAction", target, reason)

        return "Action completed on " .. target:Name()
    end
})
```

### Roleplay Command Template

```lua
ix.command.Add("RPAction", {
    description = "Perform RP action",
    arguments = {ix.type.text},
    OnRun = function(self, client, text)
        local character = client:GetCharacter()

        -- Check faction/class permissions
        if not self:CheckPermissions(character) then
            return "You cannot do this"
        end

        -- Perform action
        ix.chat.Send(client, "action", text)

        return false  -- Message sent via chat
    end,

    CheckPermissions = function(self, character)
        -- Custom permission logic
        return character:GetFaction() == FACTION_POLICE
    end
})
```

## See Also

- [Command System](../systems/commands.md) - Detailed command system reference
- [Chat System](../systems/chat.md) - Chat types and messages
- [Flag System](../systems/flags.md) - Permission flags
- Source: `gamemode/core/libs/sh_command.lua`
