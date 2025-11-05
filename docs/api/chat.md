# Chat API (ix.chat)

> **Reference**: `gamemode/core/libs/sh_chatbox.lua`

The chat API provides registration and management of chat message types with different properties, formats, colors, and hearing ranges. All roleplay communication uses the chat system.

## ⚠️ Important: Use Built-in Helix Chat Functions

**Always use Helix's built-in chat system** rather than creating custom chat handlers. The framework automatically provides:
- Multiple chat types (IC, OOC, LOOC, /me, /it, whisper, yell, etc.)
- Distance-based message hearing
- Automatic message formatting and capitalization
- Color and format customization per chat type
- Typing indicators above players
- Network synchronization
- Command integration (prefixes like `/me` auto-register as commands)
- Hook integration for message filtering

## Core Concepts

### What are Chat Classes?

Chat classes define message types with different properties:
- **IC (In-Character)**: Normal roleplay speech with limited range
- **OOC (Out-of-Character)**: Global server-wide chat
- **LOOC (Local OOC)**: Local out-of-character chat
- **/me**: Roleplay actions (e.g., "** John picks up the can")
- **/it**: Environmental actions (e.g., "** The door creaks open")
- **Whisper**: Quiet IC speech with shorter range
- **Yell**: Loud IC speech with longer range

Each chat class can have custom:
- Prefix (what player types to use it)
- Color and format
- Hearing range/conditions
- Usage restrictions
- Display behavior

### Key Terms

**Chat Class**: A registered message type with specific properties
**Prefix**: What players type to use a chat class (e.g., `/me`, `//`)
**Speaker**: Player sending the message
**Listener**: Player(s) who can hear/see the message
**Anonymous**: Message sent without showing speaker name

## Library Tables

### ix.chat.classes

**Reference**: `gamemode/core/libs/sh_chatbox.lua:22`

**Realm**: Shared

Table of all registered chat classes.

```lua
-- Access chat class data
local icChat = ix.chat.classes["ic"]
print("IC format:", icChat.format)  -- "%s says \"%s\""

-- Iterate all chat classes
for id, class in pairs(ix.chat.classes) do
    print("Chat class:", id, "Color:", class.color)
end
```

**Note**: Access this in `InitializedChatClasses` hook to ensure all classes are loaded.

## Library Functions

### ix.chat.Register

**Reference**: `gamemode/core/libs/sh_chatbox.lua:112`

**Realm**: Shared

```lua
ix.chat.Register(chatType, data)
```

Registers a new chat type with custom properties.

**Parameters**:
- `chatType` (string) - Unique identifier for chat class
- `data` (table) - Chat class configuration:
  - `prefix` (string/table, optional) - Prefix(es) players type (e.g., `/me`, `{"//", "/OOC"}`)
  - `noSpaceAfter` (bool, optional) - Allow prefix without space (e.g., `//text`)
  - `description` (string, optional) - Description shown in chatbox
  - `format` (string, optional) - Message format with `%s` placeholders (default: `"%s: \"%s\""`)
  - `color` (Color, optional) - Message color (default: `Color(242, 230, 160)`)
  - `GetColor` (function, optional) - Dynamic color function
  - `indicator` (string, optional) - Typing indicator language key (default: `"chatTyping"`)
  - `bNoIndicator` (bool, optional) - Disable typing indicator
  - `font` (string, optional) - Font for messages (default: `"ixChatFont"`)
  - `deadCanChat` (bool, optional) - Allow dead players to use
  - `CanHear` (number/function) - Hearing range in units or custom function
  - `CanSay` (function, optional) - Check if player can send message
  - `OnChatAdd` (function, optional) - Custom message display function

**Example - Simple Radio Chat**:
```lua
-- Register in InitializedChatClasses hook
hook.Add("InitializedChatClasses", "MyRadio", function()
    ix.chat.Register("radio", {
        format = "[RADIO] %s: %s",
        color = Color(100, 200, 100),
        prefix = "/r",
        description = "Speak on your radio",
        indicator = "chatTalking",
        CanHear = function(self, speaker, listener)
            -- Only faction members with radios can hear
            local speakerChar = speaker:GetCharacter()
            local listenerChar = listener:GetCharacter()

            if not (speakerChar and listenerChar) then
                return false
            end

            -- Same faction only
            return speakerChar:GetFaction() == listenerChar:GetFaction()
        end,
        CanSay = function(self, speaker, text)
            -- Must have radio item
            local char = speaker:GetCharacter()
            local inv = char:GetInventory()

            if not inv:HasItem("radio") then
                speaker:Notify("You need a radio!")
                return false
            end

            return true
        end
    })
end)
```

**Example - Faction-Specific Chat**:
```lua
hook.Add("InitializedChatClasses", "PoliceRadio", function()
    ix.chat.Register("policeradio", {
        format = "[POLICE RADIO] %s: %s",
        color = Color(50, 100, 200),
        prefix = {"/pr", "/policeradio"},
        description = "Police radio channel",
        CanHear = function(self, speaker, listener)
            local char = listener:GetCharacter()
            return char and char:GetFaction() == FACTION_POLICE
        end,
        CanSay = function(self, speaker, text)
            local char = speaker:GetCharacter()
            if not char or char:GetFaction() ~= FACTION_POLICE then
                speaker:Notify("You're not in the police force!")
                return false
            end
            return true
        end
    })
end)
```

**Example - Proximity-Based Whisper**:
```lua
hook.Add("InitializedChatClasses", "QuietWhisper", function()
    ix.chat.Register("quietwhisper", {
        format = "%s quietly whispers \"%s\"",
        GetColor = function(self, speaker, text)
            -- Darker than normal IC
            local baseColor = ix.config.Get("chatColor")
            return Color(
                math.max(baseColor.r - 50, 0),
                math.max(baseColor.g - 50, 0),
                math.max(baseColor.b - 50, 0)
            )
        end,
        CanHear = 70,  -- Very short range (70 units)
        prefix = {"/qw", "/quietwhisper"},
        description = "Whisper very quietly",
        indicator = "chatWhispering"
    })
end)
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't use print/chat.AddText directly for IC messages
hook.Add("PlayerSay", "CustomChat", function(client, text)
    PrintMessage(HUD_PRINTTALK, client:Name() .. ": " .. text)
    return ""  -- This bypasses Helix chat system!
end)

-- CORRECT: Use ix.chat.Register
ix.chat.Register("custom", {...})
```

### ix.chat.Send

**Reference**: `gamemode/core/libs/sh_chatbox.lua:303` (server), `sh_chatbox.lua:361` (client)

**Realm**: Shared

```lua
ix.chat.Send(speaker, chatType, text, bAnonymous, receivers, data)
```

Sends a chat message using specified chat type.

**Parameters**:
- `speaker` (Player) - Player sending message (can be nil for system messages)
- `chatType` (string) - Chat class ID to use
- `text` (string) - Message text
- `bAnonymous` (bool, optional) - Hide speaker's name
- `receivers` (table, optional) - Specific players to send to (server only)
- `data` (table, optional) - Additional data passed to OnChatAdd

**Example - Server**:
```lua
-- Normal IC message
ix.chat.Send(client, "ic", "Hello everyone!")

-- Anonymous message
ix.chat.Send(client, "ic", "A mysterious voice speaks", true)

-- System event message
ix.chat.Send(nil, "event", "The sirens begin to wail!")

-- Message to specific players
local nearbyPlayers = ents.FindInSphere(client:GetPos(), 500)
ix.chat.Send(client, "radio", "I need backup!", false, nearbyPlayers)

-- With custom data
ix.chat.Send(client, "roll", "75", nil, nil, {max = 100})
```

**Example - Client**:
```lua
-- Display message in local chatbox
ix.chat.Send(speaker, "ic", "This is shown locally only", false)
```

**Note**: On server, message auto-sends to appropriate receivers based on CanHear. On client, message displays locally.

### ix.chat.Parse

**Reference**: `gamemode/core/libs/sh_chatbox.lua:207`

**Realm**: Shared

```lua
local chatType, message, anonymous = ix.chat.Parse(client, message, bNoSend)
```

Parses a message to detect chat type from prefix and optionally sends it.

**Parameters**:
- `client` (Player) - Player sending message
- `message` (string) - Raw message with prefix
- `bNoSend` (bool, optional) - Don't send message, just parse

**Returns**:
- `chatType` (string) - Detected chat class ID
- `message` (string) - Message with prefix removed
- `anonymous` (bool) - Whether message is anonymous

**Example**:
```lua
-- Parse message and detect chat type
local chatType, msg, anon = ix.chat.Parse(client, "/me picks up the can", true)
print(chatType)  -- "me"
print(msg)       -- "picks up the can"

-- Parse and auto-send
ix.chat.Parse(client, "Hello!")  -- Sends as IC chat
```

**Note**: Called automatically by framework. Rarely needs manual use.

### ix.chat.Format

**Reference**: `gamemode/core/libs/sh_chatbox.lua:281`

**Realm**: Shared

```lua
local formatted = ix.chat.Format(text)
```

Formats text with basic grammar: capitalizes first letter, adds punctuation if missing, trims whitespace.

**Parameters**:
- `text` (string) - Text to format

**Returns**: (string) Formatted text

**Example**:
```lua
print(ix.chat.Format("hello"))           -- "Hello."
print(ix.chat.Format("hello world"))     -- "Hello world."
print(ix.chat.Format("wow!"))            -- "Wow!"
print(ix.chat.Format("  spaced out  "))  -- "Spaced out."
```

**Note**: Applied automatically when `chatAutoFormat` config is enabled.

## Chat Class Structure

### Complete Chat Class Example

```lua
ix.chat.Register("shout", {
    -- Prefix players type to use this chat
    prefix = {"/shout", "/s"},

    -- Allow "/shoutHello" without space
    noSpaceAfter = false,

    -- Description shown in chatbox help
    description = "Shout loudly to distant players",

    -- Format: first %s = name, second %s = message
    format = "%s shouts \"%s\"",

    -- Static color
    color = Color(255, 200, 100),

    -- OR dynamic color based on conditions
    GetColor = function(self, speaker, text)
        if speaker:Health() < 50 then
            return Color(255, 100, 100)  -- Red if injured
        end
        return Color(255, 200, 100)
    end,

    -- Typing indicator language key
    indicator = "chatYelling",

    -- Hide typing indicator
    bNoIndicator = false,

    -- Font for messages
    font = "ixChatFont",

    -- Allow dead players to use
    deadCanChat = false,

    -- Hearing range in units
    CanHear = 1000,

    -- OR custom hearing function
    CanHear = function(self, speaker, listener, data)
        local dist = speaker:GetPos():Distance(listener:GetPos())
        return dist <= 1000
    end,

    -- Check if player can send message
    CanSay = function(self, speaker, text, data)
        if not speaker:Alive() then
            speaker:Notify("You cannot shout while dead!")
            return false
        end

        if speaker:GetCharacter():GetData("muted") then
            speaker:Notify("You are muted!")
            return false
        end

        return true
    end,

    -- Custom message display (advanced)
    OnChatAdd = function(self, speaker, text, bAnonymous, data)
        local name = bAnonymous and "Someone" or speaker:Name()
        local color = self.color

        chat.AddText(
            color,
            ">> ",
            Color(255, 255, 255),
            name,
            color,
            " shouts: ",
            text,
            " <<"
        )
    end
})
```

## Complete Examples

### Admin Broadcast System

```lua
hook.Add("InitializedChatClasses", "AdminBroadcast", function()
    ix.chat.Register("broadcast", {
        format = "[BROADCAST] %s",
        color = Color(255, 150, 0),
        CanHear = 999999,  -- Global
        OnChatAdd = function(self, speaker, text)
            -- Custom display with sound
            chat.AddText(self.color, "[BROADCAST] ", color_white, text)
            surface.PlaySound("ambient/alarms/warningbell1.wav")
        end
    })
end)

-- Admin command to use it
ix.command.Add("Broadcast", {
    description = "Send a server-wide broadcast",
    adminOnly = true,
    arguments = ix.type.text,
    OnRun = function(self, client, message)
        ix.chat.Send(nil, "broadcast", message)
    end
})
```

### Item-Based Radio System

```lua
-- Radio item
ITEM.name = "Handheld Radio"
ITEM.model = "models/props_lab/citizenradio.mdl"
ITEM.description = "A portable radio for communication"

function ITEM:OnEquipped(client)
    client:SetNWBool("hasRadio", true)
end

function ITEM:OnUnequipped(client)
    client:SetNWBool("hasRadio", false)
end

-- Radio chat class
hook.Add("InitializedChatClasses", "RadioChat", function()
    ix.chat.Register("radio", {
        format = "[RADIO] %s: %s",
        color = Color(100, 200, 100),
        prefix = {"/r", "/radio"},
        description = "Speak on your radio",
        CanHear = function(self, speaker, listener)
            -- Both must have radio equipped
            return speaker:GetNWBool("hasRadio") and
                   listener:GetNWBool("hasRadio")
        end,
        CanSay = function(self, speaker, text)
            if not speaker:GetNWBool("hasRadio") then
                speaker:Notify("You need a radio equipped!")
                return false
            end
            return true
        end
    })
end)
```

### Roleplay Action Modifiers

```lua
-- Quiet action
hook.Add("InitializedChatClasses", "QuietMe", function()
    ix.chat.Register("meq", {
        format = "* %s %s",  -- Single asterisk
        GetColor = ix.chat.classes.me.GetColor,
        CanHear = ix.config.Get("chatRange", 280) * 0.5,  -- Half /me range
        prefix = {"/meq", "/qme"},
        description = "Perform a quiet action",
        indicator = "chatPerforming"
    })
end)

-- Loud action
hook.Add("InitializedChatClasses", "LoudMe", function()
    ix.chat.Register("mel", {
        format = "*** %s %s",  -- Triple asterisk
        GetColor = function(self, speaker, text)
            local baseColor = ix.chat.classes.me:GetColor(speaker, text)
            return Color(
                math.min(baseColor.r + 30, 255),
                math.min(baseColor.g + 30, 255),
                math.min(baseColor.b + 30, 255)
            )
        end,
        CanHear = ix.config.Get("chatRange", 280) * 3,
        prefix = {"/mel", "/lme"},
        description = "Perform a loud action",
        indicator = "chatPerforming"
    })
end)
```

### Language Barrier System

```lua
-- Chat class for foreign language
hook.Add("InitializedChatClasses", "ForeignLanguage", function()
    ix.chat.Register("german", {
        format = "%s says something in German: \"%s\"",
        GetColor = ix.chat.classes.ic.GetColor,
        CanHear = ix.config.Get("chatRange", 280),
        prefix = {"/german", "/de"},
        description = "Speak in German",
        OnChatAdd = function(self, speaker, text, bAnonymous, data)
            local listener = LocalPlayer()
            local listenerChar = listener:GetCharacter()

            -- Check if listener knows German
            local knowsGerman = listenerChar and
                               listenerChar:GetData("languages", {})"german"]

            local displayText = knowsGerman and text or
                              string.rep("*", #text)  -- Scramble if unknown

            local color = self:GetColor(speaker, text)
            local name = speaker:Name()

            chat.AddText(color, string.format(self.format, name, displayText))
        end
    })
end)
```

## Best Practices

### ✅ DO

- Register chat classes in `InitializedChatClasses` hook
- Use descriptive chat type names (lowercase)
- Provide multiple prefix options for convenience
- Use `CanHear` functions for complex hearing logic
- Use `CanSay` to validate speaker permissions
- Leverage existing chat classes with `ix.chat.classes.ic.GetColor`
- Test chat ranges in-game for balance
- Use language keys for descriptions

### ❌ DON'T

- Don't override PlayerSay hook for custom chat
- Don't use print/PrintMessage for IC messages
- Don't forget realm restrictions (Send server only)
- Don't make hearing ranges too large (lag)
- Don't skip CanSay validation
- Don't hardcode colors - use GetColor for flexibility
- Don't register chat classes outside hooks
- Don't use generic names like "custom" or "new"

## Common Patterns

### Dynamic Hearing Based on Attribute

```lua
CanHear = function(self, speaker, listener)
    local speakerChar = speaker:GetCharacter()
    local listenerChar = listener:GetCharacter()

    if not (speakerChar and listenerChar) then
        return false
    end

    -- Louder voice if high strength
    local speakerStr = speakerChar:GetAttribute("str", 0)
    local baseRange = ix.config.Get("chatRange", 280)
    local bonusRange = speakerStr * 5  -- +5 units per STR

    local dist = speaker:GetPos():Distance(listener:GetPos())
    return dist <= (baseRange + bonusRange)
end
```

### Cooldown on Chat Type

```lua
CanSay = function(self, speaker, text)
    local lastUse = speaker.lastShoutTime or 0
    local cooldown = 10  -- 10 second cooldown

    if CurTime() - lastUse < cooldown then
        speaker:Notify("You must wait " ..
            math.ceil(cooldown - (CurTime() - lastUse)) ..
            " seconds before shouting again!")
        return false
    end

    speaker.lastShoutTime = CurTime()
    return true
end
```

### Context-Aware Colors

```lua
GetColor = function(self, speaker, text)
    -- Red if injured
    if speaker:Health() < 30 then
        return Color(255, 100, 100)
    end

    -- Blue if underwater
    if speaker:WaterLevel() >= 3 then
        return Color(100, 150, 255)
    end

    -- Normal color
    return Color(242, 230, 160)
end
```

## Common Issues

### Chat Messages Not Showing

**Cause**: CanHear returning false or wrong receivers.

**Fix**: Debug CanHear function:
```lua
CanHear = function(self, speaker, listener)
    local result = listener:GetPos():Distance(speaker:GetPos()) <= 280
    print("Can hear?", speaker, "->", listener, result)
    return result
end
```

### Prefix Not Working

**Cause**: Missing `/` or registered after initialization.

**Fix**: Register in hook with `/` prefix:
```lua
hook.Add("InitializedChatClasses", "MyChat", function()
    ix.chat.Register("custom", {
        prefix = "/custom",  -- Must start with /
        -- ...
    })
end)
```

### Colors Not Applying

**Cause**: OnChatAdd override without using color.

**Fix**: Use GetColor or self.color in OnChatAdd:
```lua
OnChatAdd = function(self, speaker, text)
    local color = self.GetColor and self:GetColor(speaker, text) or self.color
    chat.AddText(color, text)
end
```

## Related Hooks

### InitializedChatClasses

Called after default chat classes are registered.

```lua
hook.Add("InitializedChatClasses", "MyChat", function()
    -- Register custom chat classes here
    ix.chat.Register("custom", {...})
end)
```

### PlayerMessageSend

Called when player sends a chat message (can modify).

```lua
hook.Add("PlayerMessageSend", "ModifyMessage", function(client, chatType, text, anonymous)
    -- Censor bad words
    text = string.gsub(text, "badword", "****")

    -- Return modified text
    return text
end)
```

### MessageReceived

Called on client when receiving chat message.

```lua
hook.Add("MessageReceived", "LogMessages", function(client, info)
    -- info.chatType, info.text, info.anonymous, info.data
    print("Received", info.chatType, "from", client:Name())
end)
```

### GetCharacterName

Called to get character name for chat display.

```lua
hook.Add("GetCharacterName", "CustomNames", function(speaker, chatType)
    if chatType == "radio" then
        return "Unit-" .. speaker:GetCharacter():GetID()
    end
end)
```

## See Also

- [Chat System Guide](../systems/chat.md) - User guide for chat
- [Command API](command.md) - Related command system
- [Notice API](notice.md) - Player notifications
- [Language API](language.md) - Localization for chat
- Source: `gamemode/core/libs/sh_chatbox.lua`
