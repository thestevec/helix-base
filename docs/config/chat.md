# Chat Configuration

> **Reference**: `gamemode/config/sh_config.lua:44-68`

Configuration options for the chat system, including in-character chat range, OOC settings, colors, and message formatting.

## ⚠️ Important: Use Built-in Config System

**Always use `ix.config.Get()` for chat settings** rather than hardcoding chat behavior. The framework provides:
- Centralized chat configuration
- Admin-adjustable chat ranges and delays
- Automatic color management
- Real-time config updates
- Spam prevention through delay settings

## In-Character Chat

### chatRange

**Reference**: `gamemode/config/sh_config.lua:47`

```lua
ix.config.Add("chatRange", 280, "The maximum distance a person's IC chat message goes to.", nil, {
    data = {min = 10, max = 5000, decimals = 1},
    category = "chat"
})
```

**Type**: Number (Float)
**Default**: `280`
**Range**: 10 - 5000 units
**Description**: Maximum distance in Hammer units that in-character chat can be heard

**Usage**:
```lua
-- Check if player can hear IC message
function PLUGIN:CanPlayerHearChat(speaker, listener)
    local distance = speaker:GetPos():Distance(listener:GetPos())
    local chatRange = ix.config.Get("chatRange", 280)

    return distance <= chatRange
end
```

---

### chatMax

**Reference**: `gamemode/config/sh_config.lua:51`

```lua
ix.config.Add("chatMax", 256, "The maximum amount of characters that can be sent in chat.", nil, {
    data = {min = 32, max = 1024},
    category = "chat"
})
```

**Type**: Number (Integer)
**Default**: `256`
**Range**: 32 - 1024 characters
**Description**: Maximum length of a single chat message

**Usage**:
```lua
-- Validate message length
local message = "Hello world"
local maxLen = ix.config.Get("chatMax", 256)

if #message > maxLen then
    client:Notify("Message too long! Maximum " .. maxLen .. " characters")
    return false
end
```

---

### chatAutoFormat

**Reference**: `gamemode/config/sh_config.lua:44`

```lua
ix.config.Add("chatAutoFormat", true, "Whether or not to automatically capitalize and punctuate in-character text.", nil, {
    category = "Chat"
})
```

**Type**: Boolean
**Default**: `true`
**Description**: Automatically capitalizes first letter and adds punctuation to IC messages

**Usage**:
```lua
-- Check if auto-formatting is enabled
if ix.config.Get("chatAutoFormat", true) then
    -- Message will be auto-formatted
    -- "hello world" becomes "Hello world."
end
```

---

### chatColor

**Reference**: `gamemode/config/sh_config.lua:55`

```lua
ix.config.Add("chatColor", Color(255, 255, 150), "The default color for IC chat.", nil, {category = "chat"})
```

**Type**: Color
**Default**: `Color(255, 255, 150)` (Light yellow)
**Description**: Default color for in-character chat messages

**Usage**:
```lua
-- Get IC chat color
local color = ix.config.Get("chatColor", Color(255, 255, 150))

-- Use in custom chat
ix.chat.Send(client, "ic", message, false, nil, {
    color = color
})
```

---

### chatListenColor

**Reference**: `gamemode/config/sh_config.lua:56`

```lua
ix.config.Add("chatListenColor", Color(175, 255, 150), "The color for IC chat if you are looking at the speaker.", nil, {
    category = "chat"
})
```

**Type**: Color
**Default**: `Color(175, 255, 150)` (Light green)
**Description**: Color for IC messages when you're looking directly at the speaker

**Usage**:
```lua
-- Determine chat color based on listener's view
function GetChatColor(listener, speaker)
    local trace = listener:GetEyeTrace()

    if trace.Entity == speaker then
        return ix.config.Get("chatListenColor", Color(175, 255, 150))
    end

    return ix.config.Get("chatColor", Color(255, 255, 150))
end
```

---

## Out-of-Character Chat

### allowGlobalOOC

**Reference**: `gamemode/config/sh_config.lua:63`

```lua
ix.config.Add("allowGlobalOOC", true, "Whether or not Global OOC is enabled.", nil, {
    category = "chat"
})
```

**Type**: Boolean
**Default**: `true`
**Description**: Enables or disables global out-of-character chat

**Usage**:
```lua
-- Check if global OOC is allowed
function PLUGIN:CanPlayerUseChat(client, chatType)
    if chatType == "ooc" and not ix.config.Get("allowGlobalOOC", true) then
        client:Notify("Global OOC is disabled")
        return false
    end

    return true
end
```

---

### oocDelay

**Reference**: `gamemode/config/sh_config.lua:59`

```lua
ix.config.Add("oocDelay", 10, "The delay before a player can use OOC chat again in seconds.", nil, {
    data = {min = 0, max = 10000},
    category = "chat"
})
```

**Type**: Number (Integer)
**Default**: `10`
**Range**: 0 - 10000 seconds
**Description**: Cooldown in seconds between OOC messages to prevent spam

**Usage**:
```lua
-- Track last OOC time
client.ixNextOOC = client.ixNextOOC or 0

function PLUGIN:CanPlayerUseOOC(client)
    local delay = ix.config.Get("oocDelay", 10)
    local time = CurTime()

    if time < client.ixNextOOC then
        local remaining = math.ceil(client.ixNextOOC - time)
        client:Notify("Please wait " .. remaining .. " seconds before using OOC again")
        return false
    end

    -- Set next allowed time
    client.ixNextOOC = time + delay
    return true
end
```

---

### loocDelay

**Reference**: `gamemode/config/sh_config.lua:66`

```lua
ix.config.Add("loocDelay", 0, "The delay before a player can use LOOC chat again in seconds.", nil, {
    data = {min = 0, max = 10000},
    category = "chat"
})
```

**Type**: Number (Integer)
**Default**: `0` (no delay)
**Range**: 0 - 10000 seconds
**Description**: Cooldown in seconds between LOOC (local OOC) messages

**Usage**:
```lua
-- Similar to OOC delay, but for LOOC
client.ixNextLOOC = client.ixNextLOOC or 0

function PLUGIN:CanPlayerUseLOOC(client)
    local delay = ix.config.Get("loocDelay", 0)
    if delay == 0 then return true end

    local time = CurTime()
    if time < client.ixNextLOOC then
        return false
    end

    client.ixNextLOOC = time + delay
    return true
end
```

---

## Complete Example: Chat System Integration

```lua
-- Custom chat command with config validation
ix.command.Add("Whisper", {
    description = "Whisper to nearby players",
    arguments = ix.type.text,
    OnRun = function(self, client, message)
        -- Validate message length
        local maxLen = ix.config.Get("chatMax", 256)
        if #message > maxLen then
            return "@chatMessageTooLong"
        end

        -- Get whisper range (half of normal chat range)
        local chatRange = ix.config.Get("chatRange", 280)
        local whisperRange = chatRange * 0.5

        -- Get chat color
        local color = ix.config.Get("chatColor", Color(255, 255, 150))
        local whisperColor = Color(color.r * 0.7, color.g * 0.7, color.b * 0.7)

        -- Send to nearby players
        local position = client:GetPos()
        for _, target in player.Iterator() do
            if target:GetPos():Distance(position) <= whisperRange then
                ix.chat.Send(client, "whisper", message, false, {target}, {
                    color = whisperColor
                })
            end
        end
    end
})

-- Auto-formatting implementation
function PLUGIN:FormatMessage(message)
    if not ix.config.Get("chatAutoFormat", true) then
        return message
    end

    -- Capitalize first letter
    message = string.upper(string.sub(message, 1, 1)) .. string.sub(message, 2)

    -- Add period if no punctuation
    local lastChar = string.sub(message, -1)
    if not string.match(lastChar, "[.!?]") then
        message = message .. "."
    end

    return message
end

-- Dynamic chat range based on character state
function PLUGIN:GetChatRange(speaker, chatType)
    if chatType != "ic" then return nil end

    local baseRange = ix.config.Get("chatRange", 280)
    local character = speaker:GetCharacter()

    -- Modify range based on character state
    if character:GetData("whispering") then
        return baseRange * 0.3
    elseif character:GetData("yelling") then
        return baseRange * 1.5
    end

    return baseRange
end

-- OOC spam prevention
function PLUGIN:CanPlayerUseOOC(client)
    if not ix.config.Get("allowGlobalOOC", true) then
        return false, "Global OOC is disabled"
    end

    local delay = ix.config.Get("oocDelay", 10)
    if delay == 0 then return true end

    client.ixNextOOC = client.ixNextOOC or 0
    local remaining = client.ixNextOOC - CurTime()

    if remaining > 0 then
        return false, "Please wait " .. math.ceil(remaining) .. " seconds"
    end

    client.ixNextOOC = CurTime() + delay
    return true
end
```

## Best Practices

### ✅ DO

- Use `ix.config.Get()` for all chat-related checks
- Validate message length against `chatMax`
- Check `allowGlobalOOC` before allowing OOC chat
- Use configured colors for consistency
- Implement delay checks to prevent spam
- Respect chat ranges for proximity chat
- Apply auto-formatting when enabled

### ❌ DON'T

- Don't hardcode chat ranges or distances
- Don't bypass OOC delays for specific players without good reason
- Don't ignore `chatMax` limit
- Don't use custom colors instead of config colors
- Don't disable auto-format without checking config
- Don't allow chat to exceed configured ranges
- Don't forget to validate chat permissions

## Common Patterns

### Pattern 1: Range-Based Chat

```lua
-- Send message to players in range
function SendRangedMessage(speaker, message, range)
    range = range or ix.config.Get("chatRange", 280)
    local position = speaker:GetPos()

    for _, client in player.Iterator() do
        if client:GetPos():Distance(position) <= range then
            client:ChatPrint(message)
        end
    end
end
```

### Pattern 2: Chat Type Registration with Config

```lua
-- Register custom chat type that respects config
ix.chat.Register("yell", {
    OnGetColor = function(self, speaker, text)
        return ix.config.Get("chatColor", Color(255, 255, 150))
    end,
    OnCanHear = function(self, speaker, listener)
        local range = ix.config.Get("chatRange", 280) * 2 -- Yelling is louder
        return speaker:GetPos():Distance(listener:GetPos()) <= range
    end,
    OnChatAdd = function(self, speaker, text)
        return string.upper(text) -- Yelling is uppercase
    end
})
```

### Pattern 3: Anti-Spam System

```lua
-- Generic delay system for any chat type
local chatDelays = {}

function ApplyChatDelay(client, chatType, configKey, defaultDelay)
    local delay = ix.config.Get(configKey, defaultDelay)
    if delay == 0 then return true end

    chatDelays[client] = chatDelays[client] or {}
    chatDelays[client][chatType] = chatDelays[client][chatType] or 0

    local remaining = chatDelays[client][chatType] - CurTime()

    if remaining > 0 then
        return false, math.ceil(remaining)
    end

    chatDelays[client][chatType] = CurTime() + delay
    return true
end

-- Usage
function PLUGIN:CanPlayerUseChat(client, chatType)
    if chatType == "ooc" then
        local allowed, time = ApplyChatDelay(client, "ooc", "oocDelay", 10)
        if not allowed then
            client:Notify("Wait " .. time .. " seconds")
            return false
        end
    end
end
```

## Common Issues

### Issue: Players Can't See IC Chat

**Cause**: Chat range may be too small or not respected
**Fix**: Ensure distance checks use config range

```lua
-- Always check distance against config
function PLUGIN:CanHearChat(speaker, listener)
    local distance = speaker:GetPos():Distance(listener:GetPos())
    local maxRange = ix.config.Get("chatRange", 280)

    return distance <= maxRange
end
```

### Issue: OOC Delay Not Working

**Cause**: Not tracking delay time per player
**Fix**: Store next allowed time on client object

```lua
-- Store per-player timing
client.ixNextOOC = client.ixNextOOC or 0

function CanUseOOC(client)
    local delay = ix.config.Get("oocDelay", 10)
    local time = CurTime()

    if time < client.ixNextOOC then
        return false
    end

    client.ixNextOOC = time + delay
    return true
end
```

### Issue: Chat Auto-Format Not Applied

**Cause**: Custom chat doesn't check config setting
**Fix**: Always check chatAutoFormat before formatting

```lua
-- Check before formatting
function FormatChatMessage(message)
    if ix.config.Get("chatAutoFormat", true) then
        message = string.upper(message:sub(1, 1)) .. message:sub(2)
        if not message:match("[.!?]$") then
            message = message .. "."
        end
    end

    return message
end
```

## See Also

- [Configuration System](../systems/configuration.md) - Overview of config system
- [Chat System](../systems/chat.md) - Chat system documentation
- [Commands System](../systems/commands.md) - Chat commands
- [Server Configuration](server.md) - Server-wide settings
- Source: `gamemode/config/sh_config.lua`
