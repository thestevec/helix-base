# Notice API (ix.notice / ix.util.Notify)

> **Reference**: `gamemode/core/libs/sh_notice.lua`

The notice API provides player notifications through on-screen messages or chat. Notifications appear in the top-right corner of the screen or in the chatbox depending on player preferences.

## ⚠️ Important: Use Built-in Helix Notifications

**Always use Helix's built-in notification functions** rather than chat.AddText or direct messages. The framework automatically provides:
- Prominent top-right notifications
- Chat-based notifications (player preference)
- Localized notification support
- Server-to-client notification networking
- Automatic message formatting

## Player Methods

### client:Notify

**Reference**: `gamemode/core/libs/sh_notice.lua:50` (server), `:120` (client)

**Realm**: Shared

```lua
client:Notify(message)
```

Shows notification to player.

**Parameters**:
- `message` (string) - Message to display

**Example**:
```lua
-- Simple notification
client:Notify("Welcome to the server!")

-- Error notification (ends with !)
client:Notify("You cannot do that!")

-- Success notification
client:Notify("Item purchased successfully")

-- In command
ix.command.Add("Test", {
    OnRun = function(self, client)
        client:Notify("Test command executed")
    end
})
```

### client:NotifyLocalized

**Reference**: `gamemode/core/libs/sh_notice.lua:61` (server), `:126` (client)

**Realm**: Shared

```lua
client:NotifyLocalized(phrase, ...)
```

Shows localized notification.

**Parameters**:
- `phrase` (string) - Language phrase key
- `...` - Format arguments

**Example**:
```lua
-- Localized notification
client:NotifyLocalized("charCreated")

-- With format arguments
client:NotifyLocalized("welcomeMsg", client:Name())
client:NotifyLocalized("moneyReceived", ix.currency.Get(amount))
```

### client:ChatNotify

**Reference**: `gamemode/core/libs/sh_notice.lua:68` (server), `:132` (client)

**Realm**: Shared

```lua
client:ChatNotify(message)
```

Shows notification in chatbox.

**Parameters**:
- `message` (string) - Message to display

**Example**:
```lua
-- Chat notification
client:ChatNotify("This appears in chat")

-- Error in chat (ends with !)
client:ChatNotify("Error occurred!")
```

### client:ChatNotifyLocalized

**Reference**: `gamemode/core/libs/sh_notice.lua:81` (server), `:142` (client)

**Realm**: Shared

```lua
client:ChatNotifyLocalized(phrase, ...)
```

Shows localized notification in chatbox.

**Example**:
```lua
client:ChatNotifyLocalized("cmdSuccess")
client:ChatNotifyLocalized("playerJoined", target:Name())
```

## Utility Functions

### ix.util.Notify

**Reference**: `gamemode/core/libs/sh_notice.lua:13` (server), `:93` (client)

**Realm**: Shared

```lua
ix.util.Notify(message, recipient)
```

Sends notification to player(s).

**Parameters**:
- `message` (string) - Message
- `recipient` (Player, optional) - Target player (nil = broadcast)

**Example**:
```lua
-- Notify specific player
ix.util.Notify("Hello!", client)

-- Notify everyone
ix.util.Notify("Server restarting in 5 minutes")

-- Equivalent to client:Notify()
ix.util.Notify("Test", client)
```

### ix.util.NotifyLocalized

**Reference**: `gamemode/core/libs/sh_notice.lua:29` (server), `:112` (client)

**Realm**: Shared

```lua
ix.util.NotifyLocalized(phrase, recipient, ...)
```

Sends localized notification.

**Parameters**:
- `phrase` (string) - Language phrase
- `recipient` (Player, optional) - Target player (nil = broadcast)
- `...` - Format arguments

**Example**:
```lua
-- Notify everyone with localization
ix.util.NotifyLocalized("serverRestart", nil, 5)

-- Notify specific player
ix.util.NotifyLocalized("charCreated", client)
```

## Complete Examples

### Purchase System

```lua
function PurchaseItem(client, itemID, price)
    local char = client:GetCharacter()

    if not char:HasMoney(price) then
        client:Notify("You cannot afford this item!")
        return false
    end

    char:TakeMoney(price)
    char:GetInventory():Add(itemID)

    client:NotifyLocalized("itemPurchased", ix.item.list[itemID].name)
    return true
end
```

### Server Announcement

```lua
timer.Create("HourlyAnnouncement", 3600, 0, function()
    ix.util.NotifyLocalized("hourlyReward")

    -- Give reward to all players
    for _, client in ipairs(player.GetAll()) do
        local char = client:GetCharacter()
        if char then
            char:GiveMoney(100)
        end
    end
end)
```

### Error Handling

```lua
ix.command.Add("GiveItem", {
    adminOnly = true,
    arguments = {ix.type.player, ix.type.string},
    OnRun = function(self, client, target, itemID)
        local char = target:GetCharacter()

        if not char then
            client:Notify("Target has no character!")
            return
        end

        if not ix.item.list[itemID] then
            client:Notify("Invalid item ID!")
            return
        end

        if not char:GetInventory():Add(itemID) then
            client:Notify("Target's inventory is full!")
            return
        end

        client:NotifyLocalized("itemGiven", itemID, target:Name())
        target:NotifyLocalized("itemReceived", itemID)
    end
})
```

## Best Practices

### ✅ DO

- Use Notify for quick feedback
- End error messages with "!"
- Use NotifyLocalized for multi-language support
- Use ChatNotify for longer messages
- Provide clear, concise notifications

### ❌ DON'T

- Don't spam notifications
- Don't use for debug messages (use print)
- Don't forget localization for public servers
- Don't show notifications for every small action

## Common Patterns

### Conditional Notification

```lua
if success then
    client:Notify("Operation successful")
else
    client:Notify("Operation failed!")
end
```

### Broadcast to Faction

```lua
for _, ply in ipairs(player.GetAll()) do
    local char = ply:GetCharacter()
    if char and char:GetFaction() == FACTION_POLICE then
        ply:NotifyLocalized("policeAlert")
    end
end
```

## See Also

- [Language API](language.md) - Localization for notices
- [Chat API](chat.md) - Chat-based notifications
- [Command API](command.md) - Command feedback
- Source: `gamemode/core/libs/sh_notice.lua`
