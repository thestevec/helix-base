# Complete Command Examples

> **Reference**: `gamemode/core/libs/sh_command.lua`, `plugins/doors/sh_commands.lua`

This document provides complete, working examples of Helix commands from simple to complex.

## ⚠️ Important: Use Built-in Command System

**Always use `ix.command.Add()`** rather than creating custom chat hooks. The framework provides:
- Automatic argument type validation
- Permission checking (admin, superadmin, flags)
- Tab completion for player names
- Help system integration
- Cooldown support
- Multiple command prefixes (`/`, `!`, `@`)

## Example 1: Simple Command

The simplest command with no arguments.

### Complete Code

```lua
ix.command.Add("Time", {
    description = "Display the current server time",
    OnRun = function(self, client)
        local timeString = os.date("%I:%M:%S %p")
        return "Server time: " .. timeString
    end
})
```

### Usage

```
/time
Output: "Server time: 02:45:30 PM"
```

### Key Points

- **`return "message"`** sends feedback to the player
- **`self`** is the command table
- **`client`** is the player who ran the command
- No arguments needed for simple info commands

## Example 2: Command with Arguments

Command that takes a player and amount.

### Complete Code

```lua
ix.command.Add("GiveMoney", {
    description = "Give money to a player",
    adminOnly = true,
    arguments = {
        ix.type.player,  -- First argument: target player
        ix.type.number   -- Second argument: amount
    },
    OnRun = function(self, client, target, amount)
        -- Validate amount
        if amount <= 0 then
            return "Amount must be greater than 0"
        end

        if amount > 10000 then
            return "Maximum amount is 10,000"
        end

        -- Get target's character
        local character = target:GetCharacter()

        if !character then
            return target:Name() .. " does not have a character"
        end

        -- Give money
        character:GiveMoney(amount)

        -- Notify target
        target:Notify("You received $" .. amount .. " from an admin")

        -- Return feedback
        return "Gave $" .. amount .. " to " .. target:Name()
    end
})
```

### Usage

```
/givemoney "John Doe" 500
Output: "Gave $500 to John Doe"
```

### Key Points

- **`adminOnly = true`**: Only admins can use this
- **`ix.type.player`**: Validates and finds player by name
- **`ix.type.number`**: Validates numeric input
- Always validate input and check for nil values

## Example 3: Optional Arguments

Command with required and optional parameters.

### Complete Code

```lua
ix.command.Add("Teleport", {
    description = "Teleport yourself or another player",
    adminOnly = true,
    arguments = {
        bit.bor(ix.type.player, ix.type.optional),  -- Optional: target player
        bit.bor(ix.type.player, ix.type.optional)   -- Optional: destination player
    },
    OnRun = function(self, client, target, destination)
        -- Case 1: No arguments - teleport to where you're looking
        if !target and !destination then
            local trace = client:GetEyeTrace()
            client:SetPos(trace.HitPos)
            return "Teleported to trace position"
        end

        -- Case 2: One argument - teleport to that player
        if target and !destination then
            client:SetPos(target:GetPos())
            return "Teleported to " .. target:Name()
        end

        -- Case 3: Two arguments - teleport target to destination
        if target and destination then
            target:SetPos(destination:GetPos())
            return "Teleported " .. target:Name() .. " to " .. destination:Name()
        end
    end
})
```

### Usage

```
/teleport                    - Teleport to where you're looking
/teleport "John Doe"         - Teleport to John Doe
/teleport "John" "Jane"      - Teleport John to Jane
```

### Key Points

- **`bit.bor(ix.type.player, ix.type.optional)`**: Makes argument optional
- Handle different cases based on which arguments are provided
- Return descriptive feedback for each case

## Example 4: Text Argument

Command that uses `ix.type.text` for multi-word input.

### Complete Code

```lua
ix.command.Add("CharSetDesc", {
    description = "Set your character's description",
    arguments = {
        ix.type.text  -- Takes rest of input as description
    },
    OnRun = function(self, client, description)
        local character = client:GetCharacter()

        if !character then
            return "You must have an active character"
        end

        -- Validate description length
        if #description < 10 then
            return "Description must be at least 10 characters"
        end

        if #description > 500 then
            return "Description cannot exceed 500 characters"
        end

        -- Filter inappropriate content (basic example)
        local lower = string.lower(description)
        if string.find(lower, "badword1") or string.find(lower, "badword2") then
            return "Description contains inappropriate content"
        end

        -- Set description
        character:SetDescription(description)

        return "Description updated successfully"
    end
})
```

### Usage

```
/charsetdesc This is my character's complete description with multiple words
Output: "Description updated successfully"
```

### Key Points

- **`ix.type.text`**: Captures all remaining text
- Always validate text input (length, content)
- Use for descriptions, reasons, messages

## Example 5: Multiple Arguments with Validation

Complex command with multiple argument types.

### Complete Code

```lua
ix.command.Add("SetHealth", {
    description = "Set a player's health and armor",
    superAdminOnly = true,
    arguments = {
        ix.type.player,  -- Target player
        ix.type.number,  -- Health amount
        bit.bor(ix.type.number, ix.type.optional),  -- Optional: armor
        bit.bor(ix.type.text, ix.type.optional)     -- Optional: reason
    },
    OnRun = function(self, client, target, health, armor, reason)
        -- Validate health
        health = math.Clamp(health, 0, 999)

        -- Set health
        target:SetHealth(health)

        local message = "Set " .. target:Name() .. "'s health to " .. health

        -- Set armor if provided
        if armor then
            armor = math.Clamp(armor, 0, 255)
            target:SetArmor(armor)
            message = message .. " and armor to " .. armor
        end

        -- Notify target
        local notification = "Your health was set to " .. health
        if armor then
            notification = notification .. " and armor to " .. armor
        end
        if reason then
            notification = notification .. " (" .. reason .. ")"
        end

        target:Notify(notification)

        -- Log action
        ix.log.Add(client, "setHealth", target:Name(), health, armor or 0, reason or "No reason")

        return message
    end
})
```

### Usage

```
/sethealth "John Doe" 100
/sethealth "John Doe" 100 50
/sethealth "John Doe" 100 50 PvP event reward
Output: "Set John Doe's health to 100 and armor to 50"
```

### Key Points

- Mix required and optional arguments
- **`math.Clamp()`**: Ensure values within valid range
- Log important admin actions
- Provide feedback to both admin and target

## Example 6: Custom Permission Check

Command with custom access control.

### Complete Code

```lua
ix.command.Add("ChangeMap", {
    description = "Change the map",
    arguments = {
        ix.type.string  -- Map name
    },
    OnCheckAccess = function(self, client)
        -- Custom permission check
        -- Allow superadmins OR players with "Map Manager" privilege
        if client:IsSuperAdmin() then
            return true
        end

        if CAMI.PlayerHasPermission(client, "Helix - Map Management") then
            return true
        end

        -- Check custom flag
        local character = client:GetCharacter()
        if character and character:HasFlags("m") then  -- 'm' for map change
            return true
        end

        return false
    end,
    OnRun = function(self, client, mapName)
        -- Validate map exists
        if !file.Exists("maps/" .. mapName .. ".bsp", "GAME") then
            return "Map '" .. mapName .. "' does not exist"
        end

        -- Announce to everyone
        for _, ply in ipairs(player.GetAll()) do
            ply:ChatPrint("Map changing to " .. mapName .. " in 10 seconds...")
        end

        -- Change map after delay
        timer.Simple(10, function()
            RunConsoleCommand("changelevel", mapName)
        end)

        return "Changing map to " .. mapName .. " in 10 seconds"
    end
})
```

### Usage

```
/changemap rp_city45_2021
Output: "Changing map to rp_city45_2021 in 10 seconds"
```

### Key Points

- **`OnCheckAccess`**: Custom permission logic
- Check multiple permission sources
- Validate file existence before executing
- Announce important actions to all players

## Example 7: Command with Cooldown

Command that limits how often it can be used.

### Complete Code

```lua
-- Store cooldowns
PLUGIN.commandCooldowns = PLUGIN.commandCooldowns or {}

ix.command.Add("Heal", {
    description = "Heal yourself (5 minute cooldown)",
    OnRun = function(self, client)
        local steamID = client:SteamID()
        local cooldown = PLUGIN.commandCooldowns[steamID]

        -- Check cooldown
        if cooldown and cooldown > CurTime() then
            local remaining = math.ceil(cooldown - CurTime())
            return "You must wait " .. remaining .. " seconds before healing again"
        end

        -- Heal player
        client:SetHealth(client:GetMaxHealth())
        client:SetArmor(100)
        client:EmitSound("items/medshot4.wav")

        -- Set cooldown (5 minutes)
        PLUGIN.commandCooldowns[steamID] = CurTime() + 300

        return "You have been healed (5 minute cooldown)"
    end
})
```

### Usage

```
/heal
Output: "You have been healed (5 minute cooldown)"

/heal (within 5 minutes)
Output: "You must wait 243 seconds before healing again"
```

### Key Points

- Store cooldowns in plugin table
- Use `CurTime()` for time tracking
- Display remaining time to user
- Clean up old cooldowns periodically

## Example 8: Command with Alias

Command with multiple names.

### Complete Code

```lua
ix.command.Add("Goto", {
    description = "Teleport to a player",
    alias = {"tp", "teleport", "go"},
    adminOnly = true,
    arguments = {
        ix.type.player
    },
    OnRun = function(self, client, target)
        if !IsValid(target) then
            return "Invalid target"
        end

        if target == client then
            return "You cannot teleport to yourself"
        end

        -- Teleport to target
        client:SetPos(target:GetPos() + Vector(0, 0, 10))

        -- Notify target
        target:Notify(client:Name() .. " teleported to you")

        return "Teleported to " .. target:Name()
    end
})
```

### Usage

```
/goto "John Doe"
/tp "John Doe"
/teleport "John Doe"
/go "John Doe"

All do the same thing!
```

### Key Points

- **`alias = {...}`**: Multiple command names
- Useful for common shortcuts
- All aliases use the same logic

## Example 9: Flag-Based Command

Command restricted to specific character flags.

### Complete Code

```lua
ix.command.Add("VendorMenu", {
    description = "Open vendor management menu",
    flag = "v",  -- Requires 'v' flag
    OnRun = function(self, client)
        -- Get character
        local character = client:GetCharacter()

        if !character then
            return "You must have a character"
        end

        -- Double-check flag (redundant but safe)
        if !character:HasFlags("v") then
            return "You don't have vendor permissions"
        end

        -- Open vendor menu
        if SERVER then
            net.Start("ixVendorMenu")
            net.Send(client)
        end

        return false  -- Don't show feedback in chat
    end
})
```

### Usage

```
/vendormenu
(Opens vendor menu if player has 'v' flag)
```

### Key Points

- **`flag = "v"`**: Requires specific character flag
- Framework automatically checks flag
- Return `false` to suppress chat feedback
- Useful for faction/class-specific commands

## Example 10: Server-Side Only Command

Command that should never run on client.

### Complete Code

```lua
-- Ensure this only exists on server
if SERVER then
    ix.command.Add("BanOffline", {
        description = "Ban a player by Steam ID",
        superAdminOnly = true,
        arguments = {
            ix.type.steamid,  -- Steam ID
            ix.type.number,   -- Ban duration (minutes)
            ix.type.text      -- Ban reason
        },
        OnRun = function(self, client, steamID, duration, reason)
            -- Validate Steam ID format
            if !string.match(steamID, "STEAM_%d:%d:%d+") then
                return "Invalid Steam ID format"
            end

            -- Check if player is online
            for _, ply in ipairs(player.GetAll()) do
                if ply:SteamID() == steamID then
                    -- Player is online, kick them
                    ply:Kick("Banned: " .. reason)
                    break
                end
            end

            -- Add to ban database
            local banID = ix.data.Get("banID", 0) + 1
            ix.data.Set("banID", banID)

            local banData = {
                steamID = steamID,
                adminSteamID = client:SteamID(),
                adminName = client:Name(),
                reason = reason,
                duration = duration,
                timestamp = os.time(),
                expires = os.time() + (duration * 60)
            }

            ix.data.Set("ban_" .. steamID, banData)

            -- Log the ban
            ix.log.Add(client, "ban", steamID, duration, reason)

            return "Banned " .. steamID .. " for " .. duration .. " minutes: " .. reason
        end
    })
end
```

### Usage

```
/banoffline STEAM_0:1:12345678 1440 Cheating
Output: "Banned STEAM_0:1:12345678 for 1440 minutes: Cheating"
```

### Key Points

- **`if SERVER`**: Only register on server
- **`ix.type.steamid`**: Validates Steam ID format
- Use for administrative commands
- Always log important actions

## ⚠️ Do NOT

```lua
-- WRONG: Don't create chat hooks
hook.Add("PlayerSay", "MyCommand", function(client, text)
    if text == "!heal" then
        -- Use ix.command.Add instead!
    end
end)

-- WRONG: Don't trust client input
ix.command.Add("Bad", {
    arguments = {ix.type.string},
    OnRun = function(self, client, path)
        file.Delete(path)  -- DANGEROUS! No validation!
    end
})

-- WRONG: Don't forget to validate
ix.command.Add("Bad2", {
    arguments = {ix.type.number},
    OnRun = function(self, client, amount)
        client:SetHealth(amount)  -- What if amount is 999999?
    end
})

-- WRONG: Don't use CLIENT-only functions on server
ix.command.Add("Bad3", {
    OnRun = function(self, client)
        LocalPlayer():ChatPrint("Hi")  -- LocalPlayer() doesn't exist on server!
    end
})
```

## Best Practices

### ✅ DO

- Use `ix.command.Add()` for all commands
- Validate all input (length, range, existence)
- Use appropriate argument types
- Check for nil values and valid entities
- Use `math.Clamp()` for numeric ranges
- Provide clear feedback messages
- Log important admin actions
- Use `adminOnly` or `superAdminOnly` for powerful commands
- Return `false` to suppress chat feedback
- Add aliases for commonly used commands

### ❌ DON'T

- Don't trust user input without validation
- Don't forget realm checks (`if SERVER`)
- Don't use expensive operations without cooldowns
- Don't forget to check if character exists
- Don't expose dangerous file/console operations
- Don't create chat hooks instead of commands
- Don't forget to handle edge cases
- Don't use CLIENT functions on SERVER

## See Also

- [Command System](../systems/commands.md) - Complete command documentation
- [Type System](../advanced/type-system.md) - Argument type details
- [Flags System](../systems/flags.md) - Character flags
- [CAMI System](../advanced/cami.md) - Advanced permissions
- Source: `gamemode/core/libs/sh_command.lua` - Command implementation
- Source: `plugins/doors/sh_commands.lua` - Real command examples
