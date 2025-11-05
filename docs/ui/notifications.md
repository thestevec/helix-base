# Notification System (ix.util.Notify)

> **Reference**: `gamemode/core/libs/sh_notice.lua`, `gamemode/core/derma/cl_notice.lua`

The notification system displays prominent messages to players in the top-right corner of their screen or in the chatbox. It supports both regular and localized messages with automatic error detection.

## ⚠️ Important: Use Built-in Notification Functions

**Always use Helix's notification system** rather than creating custom notification UI. The framework provides:
- Automatic positioning and stacking of multiple notices
- Smooth enter/exit animations
- Error highlighting (red flash for messages ending in "!")
- Automatic duration management with user preference
- Hover-to-fade interaction
- Localization support
- Network synchronization from server to client
- Chatbox fallback option

## Core Concepts

### What is the Notification System?

The notification system consists of:

1. **Notice Manager (ixNoticeManager)** - Container managing all active notices
2. **Notice Panel (ixNotice)** - Individual notification displays
3. **Notification Functions** - Server/client functions to trigger notifications

### Key Terms

- **Notice** - A temporary message displayed to the player
- **Error Notice** - Notice with red highlight (text ends in "!")
- **Localized Notice** - Translated message using language phrases
- **Chat Notice** - Alternative notification in chatbox instead of overlay
- **Duration** - How long notices display (user configurable)

## Sending Notifications

### Player:Notify()

**Reference**: `gamemode/core/libs/sh_notice.lua:50` (server), `120` (client)

Display a notification to a specific player.

```lua
-- SERVER: Notify single player
player:Notify("You picked up an item!")

-- SERVER: Error notification (ends with !)
player:Notify("You don't have permission!")

-- CLIENT: Notify local player
LocalPlayer():Notify("Action completed")
```

### Player:NotifyLocalized()

**Reference**: `gamemode/core/libs/sh_notice.lua:61` (server), `126` (client)

Display a localized notification using language phrases.

```lua
-- SERVER: Notify with localization
player:NotifyLocalized("itemPickup", "Medkit")
-- Displays: "You picked up a Medkit" (in player's language)

-- With multiple arguments
player:NotifyLocalized("mapRestarting", 10)
-- Displays: "The map will restart in 10 seconds!"

-- Error notification
player:NotifyLocalized("noPermission")
-- If phrase ends with !, displays as error
```

### ix.util.Notify()

**Reference**: `gamemode/core/libs/sh_notice.lua:13` (server), `93` (client)

Notify a player or broadcast to all players.

```lua
-- SERVER: Notify specific player
ix.util.Notify("Message", player)

-- SERVER: Broadcast to all players
ix.util.Notify("Server announcement!", nil)

-- CLIENT: Notify local player
ix.util.Notify("Local message")
```

### ix.util.NotifyLocalized()

**Reference**: `gamemode/core/libs/sh_notice.lua:29` (server), `112` (client)

Send localized notification.

```lua
-- SERVER: Notify specific player
ix.util.NotifyLocalized("phraseKey", player, arg1, arg2)

-- SERVER: Broadcast to all
ix.util.NotifyLocalized("serverRestart", nil, 60)

-- CLIENT: Notify local player
ix.util.NotifyLocalized("actionComplete")
```

## Chat Notifications

### Player:ChatNotify()

**Reference**: `gamemode/core/libs/sh_notice.lua:68` (server), `132` (client)

Display notification in chatbox instead of overlay.

```lua
-- SERVER: Chat notification
player:ChatNotify("This appears in chat")

-- Useful for less important messages
player:ChatNotify("Your stamina is regenerating...")
```

### Player:ChatNotifyLocalized()

**Reference**: `gamemode/core/libs/sh_notice.lua:81` (server), `142` (client)

Localized chat notification.

```lua
-- SERVER: Localized chat notification
player:ChatNotifyLocalized("itemDropped", itemName)
```

## User Preferences

### Notice Duration

**Reference**: `gamemode/core/derma/cl_notice.lua:46`

Players can configure how long notices display:

```lua
-- Get user's preferred duration (default: 8 seconds)
local duration = ix.option.Get("noticeDuration", 8)

-- Set duration programmatically (persists)
ix.option.Set("noticeDuration", 10)
```

### Maximum Notices

**Reference**: `gamemode/core/derma/cl_notice.lua:60`

Players can configure max simultaneous notices:

```lua
-- Get max notice count (default: 4)
local maxNotices = ix.option.Get("noticeMax", 4)

-- Set max count
ix.option.Set("noticeMax", 6)
```

### Chat Notices Mode

**Reference**: `gamemode/core/libs/sh_notice.lua:94`

Players can redirect notices to chatbox:

```lua
-- Check if player prefers chat notices
if ix.option.Get("chatNotices", false) then
    -- All notices go to chatbox
end

-- Toggle chat notices
ix.option.Set("chatNotices", true)
```

## Complete Example: Item Pickup System

```lua
-- SERVER: Notify on item pickup
function ITEM:OnPickup(character, player)
    if not self:CanPickup(player) then
        player:Notify("You can't pick this up!")
        return false
    end

    -- Success notification
    player:NotifyLocalized("itemPickup", self:GetName())

    -- Notify nearby players
    for _, ply in ipairs(ents.FindInSphere(player:GetPos(), 256)) do
        if ply != player and ply:IsPlayer() then
            ply:NotifyLocalized("itemPickupOther", player:Name(), self:GetName())
        end
    end

    return true
end
```

## Complete Example: Admin Command Feedback

```lua
-- SERVER: Command with notifications
ix.command.Add("GiveItem", {
    description = "Give an item to a player",
    adminOnly = true,
    arguments = {
        ix.type.player,
        ix.type.string
    },
    OnRun = function(self, client, target, itemID)
        local item = ix.item.list[itemID]

        if not item then
            client:Notify("Invalid item ID!")
            return
        end

        local character = target:GetCharacter()

        if not character then
            client:Notify("Target has no character!")
            return
        end

        local inventory = character:GetInventory()

        if not inventory then
            client:Notify("Target has no inventory!")
            return
        end

        inventory:Add(itemID, 1, nil, nil, function(addedItem)
            if addedItem then
                -- Notify admin
                client:NotifyLocalized("itemGiven", item.name, target:Name())

                -- Notify target
                target:NotifyLocalized("itemReceived", item.name)

                -- Log to chat
                ix.util.NotifyLocalized("adminItemGive", nil, client:Name(), item.name, target:Name())
            else
                client:Notify("Failed to give item - inventory full!")
            end
        end)
    end
})
```

## Complete Example: Timed Warnings

```lua
-- SERVER: Countdown notifications
function StartServerRestart(delay)
    local intervals = {300, 180, 60, 30, 10, 5, 4, 3, 2, 1}

    for _, seconds in ipairs(intervals) do
        if seconds <= delay then
            timer.Simple(delay - seconds, function()
                -- Broadcast warning
                ix.util.NotifyLocalized("serverRestartWarning", nil, seconds)

                -- Play sound at final countdown
                if seconds <= 5 then
                    for _, ply in ipairs(player.GetAll()) do
                        ply:EmitSound("buttons/button17.wav")
                    end
                end
            end)
        end
    end

    -- Final restart
    timer.Simple(delay, function()
        game.ConsoleCommand("changelevel " .. game.GetMap() .. "\n")
    end)
end
```

## Complete Example: Conditional Notifications

```lua
-- SERVER: Smart notification based on context
function NotifyPlayerOfStatus(player, statusType, value)
    local character = player:GetCharacter()

    if not character then
        return
    end

    -- Different notification style based on severity
    if statusType == "health" then
        if value <= 25 then
            player:Notify("Your health is critical!")  -- Red error
        elseif value <= 50 then
            player:ChatNotify("Your health is low.")  -- Chat only
        end
        -- Don't notify for high health

    elseif statusType == "stamina" then
        if value <= 10 then
            player:ChatNotify("You are exhausted.")
        end

    elseif statusType == "hunger" then
        if value >= 80 then
            player:Notify("You are starving!")
        elseif value >= 50 then
            player:ChatNotify("You are getting hungry.")
        end
    end
end

-- Hook into attribute changes
hook.Add("OnCharacterAttributeBoost", "StatusNotifications", function(character, attributeID, amount)
    local player = character:GetPlayer()

    if not IsValid(player) then
        return
    end

    local attribute = ix.attributes.list[attributeID]

    if attribute then
        player:NotifyLocalized("attributeIncreased", attribute.name, amount)
    end
end)
```

## Direct Panel Access (Advanced)

### Adding Notice Manually

**Reference**: `gamemode/core/derma/cl_notice.lua:31`

```lua
-- CLIENT: Create notice directly (rarely needed)
if IsValid(ix.gui.notices) then
    local notice = ix.gui.notices:AddNotice("Custom message", false)
    -- notice is the ixNotice panel

    -- Customize notice
    notice:SetFont("ixBigFont")
end
```

### Clearing All Notices

**Reference**: `gamemode/core/derma/cl_notice.lua:25`

```lua
-- CLIENT: Remove all notices
if IsValid(ix.gui.notices) then
    ix.gui.notices:Clear()
end
```

## Best Practices

### ✅ DO

- Use `player:Notify()` for single-player messages
- Use `player:NotifyLocalized()` for translated messages
- End error messages with "!" for red highlighting
- Use `ChatNotify()` for less important/frequent messages
- Broadcast server-wide events with `nil` recipient
- Respect user's `chatNotices` preference
- Provide context in notification text
- Use localization for all user-facing text

### ❌ DON'T

- Don't spam notifications (causes visual clutter)
- Don't send empty or meaningless messages
- Don't use notifications for debug info (use console)
- Don't forget localization for non-English speakers
- Don't bypass the notification system with custom UI
- Don't send notifications in loops without throttling
- Don't use long messages (keep under ~60 characters)
- Don't forget "!" suffix for error notifications

## Common Patterns

### Pattern 1: Success/Failure Feedback

```lua
-- Always provide clear feedback
function AttemptAction(player)
    local success, reason = CanDoAction(player)

    if success then
        -- Positive feedback
        player:NotifyLocalized("actionSuccess")
        return true
    else
        -- Error feedback
        player:Notify(reason .. "!")  -- ! makes it red
        return false
    end
end
```

### Pattern 2: Proximity Notifications

```lua
-- Notify players in radius
function NotifyNearbyPlayers(position, message, radius)
    for _, ply in ipairs(ents.FindInSphere(position, radius or 512)) do
        if ply:IsPlayer() then
            ply:Notify(message)
        end
    end
end

-- Usage
NotifyNearbyPlayers(entity:GetPos(), "The alarm is sounding!", 1024)
```

### Pattern 3: Throttled Notifications

```lua
-- Prevent notification spam
local lastNotify = {}

function ThrottledNotify(player, message, delay)
    delay = delay or 5
    local steamID = player:SteamID()
    local now = CurTime()

    if not lastNotify[steamID] or lastNotify[steamID] + delay < now then
        player:Notify(message)
        lastNotify[steamID] = now
    end
end

-- Usage in frequently-called hooks
hook.Add("PlayerTick", "StaminaWarning", function(player)
    local stamina = player:GetStamina()

    if stamina < 10 then
        ThrottledNotify(player, "You are exhausted!", 10)
    end
end)
```

## Common Issues

### Notification Not Appearing

**Cause**: Sent on client to non-local player or character menu open
**Fix**: Only notify local player on client, or use server-side

```lua
-- WRONG (client-side)
for _, ply in ipairs(player.GetAll()) do
    ply:Notify("Message")  -- Only works for LocalPlayer()!
end

-- CORRECT (server-side)
for _, ply in ipairs(player.GetAll()) do
    ply:Notify("Message")  -- Works for all players
end
```

### Localized Message Shows Key

**Cause**: Language phrase doesn't exist
**Fix**: Register phrase in language table

```lua
-- SHARED: Register language phrases
ix.lang.AddTable("english", {
    myCustomPhrase = "This is my custom notification",
    itemPickup = "You picked up %s"
})

-- Now can use:
player:NotifyLocalized("myCustomPhrase")
```

### Notification Spam

**Cause**: Sending in high-frequency hooks without throttling
**Fix**: Use throttling or move to less frequent hook

```lua
-- WRONG
hook.Add("Think", "BadNotify", function()
    if condition then
        LocalPlayer():Notify("Condition met!")  -- Spams!
    end
end)

-- CORRECT
local lastCheck = 0
hook.Add("Think", "GoodNotify", function()
    if CurTime() - lastCheck < 2 then return end
    lastCheck = CurTime()

    if condition then
        LocalPlayer():Notify("Condition met!")  -- Max once per 2 seconds
    end
end)
```

### Error Notification Not Red

**Cause**: Forgot to end message with "!"
**Fix**: Always end error messages with exclamation mark

```lua
-- WRONG
player:Notify("You don't have permission")  -- Normal color

-- CORRECT
player:Notify("You don't have permission!")  -- Red highlight
```

## See Also

- [Chat System](../systems/chat.md) - Chat notifications integration
- [Language System](../libraries/localization.md) - Localization phrases
- [Configuration System](../systems/configuration.md) - Notice user options
- [HUD System](hud.md) - On-screen display
- [Derma Overview](derma-overview.md) - UI panels
- Source: `gamemode/core/libs/sh_notice.lua`
- Source: `gamemode/core/derma/cl_notice.lua`
