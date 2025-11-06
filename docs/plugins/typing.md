# Typing Indicator Plugin

> **Reference**: `plugins/typing.lua`

The Typing Indicator plugin displays a visual indicator above players' heads when they are typing in chat. The indicator shows what type of message they're typing (IC, OOC, commands, etc.) and fades based on distance, enhancing roleplay immersion.

## ⚠️ Important: Use Built-in Typing System

**The typing indicator works automatically**. The framework provides:
- Automatic detection of chat type being typed
- Distance-based visibility and fading
- Integration with all chat types
- Smooth animations
- Network optimization (PVS only)
- Localized indicator text
- Support for custom indicators

## Core Concepts

### What is the Typing Indicator?

The typing indicator is 3D text above players showing:
- That they are typing
- What type of message (IC, OOC, etc.)
- Appears when player starts typing
- Fades out when they finish
- Visible only within chat range

### Key Features

- **Auto-Detection**: Automatically determines chat type
- **Distance Fading**: Indicator fades based on distance
- **Smooth Animations**: Fade in/out with tweening
- **Localization**: Supports multiple languages
- **Custom Indicators**: Commands and chat types can specify custom text
- **Performance**: Only sent to players in PVS (potentially visible set)

## How It Works

### Automatic Detection

**Reference**: `plugins/typing.lua:29-96`

When player types, the plugin detects:
1. **IC Chat**: No prefix, alphanumeric first character
2. **Chat Types**: Matches registered chat types (e.g., /w, /y, /me)
3. **Commands**: Matches registered commands (e.g., /roll, /event)
4. **Custom Indicators**: Uses command.indicator or chat.indicator

### Visual Display

**Reference**: `plugins/typing.lua:114-173`

Indicator appears above player's head:
- **Position**: Above head bone + 10 units
- **Font**: Large outlined font (128pt)
- **Color**: White with dark outline
- **Visibility**: Within chat range
- **Fade**: Based on distance

## Customization

### Custom Command Indicator

**Reference**: `plugins/typing.lua:81-94`

```lua
-- In your command definition
ix.command.Add("MyCommand", {
    description = "Does something",
    indicator = "commandTyping",  -- Language phrase
    bNoIndicator = false,  -- Set true to hide indicator

    OnRun = function(self, client, ...)
        -- Command logic
    end
})

-- In language file:
LANGUAGE["commandTyping"] = "typing command..."
```

### Custom Chat Type Indicator

```lua
-- In your chat type definition
ix.chat.Register("mychat", {
    format = "...format...",
    indicator = "myCustomTyping",  -- Language phrase
    bNoIndicator = false,  -- Set true to hide indicator

    OnChatAdd = function(...)
        -- Chat logic
    end
})

-- In language file:
LANGUAGE["myCustomTyping"] = "typing custom message..."
```

### GetTypingIndicator Hook

**Reference**: `plugins/typing.lua:69-96`

```lua
-- Override indicator for specific conditions
function PLUGIN:GetTypingIndicator(character, text)
    -- Return indicator name or nil

    -- Example: Custom indicator for admin commands
    if text:StartWith("/admin") then
        return "@adminCommand"  -- Uses L("adminCommand")
    end

    -- Return nil to use default detection
end
```

### GetTypingIndicatorPosition Hook

**Reference**: `plugins/typing.lua:98-112`

```lua
-- Override indicator position
function PLUGIN:GetTypingIndicatorPosition(client)
    -- Return Vector position

    -- Example: Higher position for tall models
    local modelHeight = client:OBBMaxs().z
    return client:GetPos() + Vector(0, 0, modelHeight + 20)
end
```

## Complete Examples

### Example 1: Hide Indicator for Stealth Commands

```lua
ix.command.Add("StealthAction", {
    description = "Performs stealth action",
    bNoIndicator = true,  -- Don't show typing indicator

    OnRun = function(self, client)
        -- Stealth logic
    end
})
```

### Example 2: Custom Indicator for Radio Chat

```lua
ix.chat.Register("radio", {
    format = "[RADIO] %s: \"%s\"",
    indicator = "radioTyping",  -- Custom indicator
    bNoIndicator = false,

    OnChatAdd = function(speaker, text)
        -- Radio chat logic
    end
})

-- In language file:
LANGUAGE["radioTyping"] = "using radio..."
```

### Example 3: Indicator Position for Crouching

```lua
function PLUGIN:GetTypingIndicatorPosition(client)
    local base = client:GetPos()
    local offset = client:Crouching() and 40 or 75

    return base + Vector(0, 0, offset)
end
```

### Example 4: Different Indicator for VIPs

```lua
function PLUGIN:GetTypingIndicator(character, text)
    local client = character:GetPlayer()

    if client:GetUserGroup() == "vip" and text:StartWith("!") then
        return "@vipChat"  -- Special VIP indicator
    end

    -- Use default detection
end
```

## Best Practices

### ✅ DO

- Use custom indicators for unique chat types
- Set bNoIndicator for private/hidden commands
- Keep indicator text short
- Use localization for indicator text
- Test indicators at various distances
- Consider disabling for stealth mechanics

### ❌ DON'T

- Don't make indicators too large
- Don't forget to localize indicator text
- Don't show indicators for whisper/private chat
- Don't use excessively long indicator text
- Don't override position without testing

## Common Patterns

### Pattern 1: Anonymous Chat (No Indicator)

```lua
ix.chat.Register("anon", {
    format = "[Anonymous] %s",
    bNoIndicator = true,  -- Don't reveal who's typing
    -- ...
})
```

### Pattern 2: Command-Specific Indicator

```lua
ix.command.Add("Roll", {
    description = "Roll dice",
    indicator = "rolling",

    OnRun = function(self, client)
        -- Roll logic
    end
})

LANGUAGE["rolling"] = "rolling..."
```

### Pattern 3: Proximity-Based Visibility

The plugin automatically handles this - indicators only show within chat range.

## Common Issues

### Indicator Not Showing

**Cause**: bNoIndicator set to true or out of range
**Fix**: Check chat/command definition and distance

### Wrong Indicator Text

**Cause**: Missing language phrase
**Fix**: Add phrase to language file

```lua
LANGUAGE["chatTyping"] = "typing..."
```

### Indicator in Wrong Position

**Cause**: Model has no head bone
**Fix**: Override GetTypingIndicatorPosition

```lua
function PLUGIN:GetTypingIndicatorPosition(client)
    return client:EyePos() + Vector(0, 0, 10)
end
```

### Indicator Stays After Typing

**Cause**: Network issue or animation disabled
**Fix**: Usually resolves automatically, check animations option

## Technical Details

### Network Protocol

**Reference**: `plugins/typing.lua:214-243`

- **Network String**: `ixTypeClass`
- **Rate Limit**: 0.2 seconds between updates
- **Optimization**: Only sent to players in PVS (potentially visible)
- **Broadcast**: End messages broadcast to ensure cleanup

### Animation System

**Reference**: `plugins/typing.lua:136-152, 195-211`

With animations enabled:
- **Fade In**: 0.5 seconds, outCubic easing
- **Fade Out**: 0.5 seconds, inCubic easing
- **Property**: `ixChatClassAnimation` (0 to 1)

Without animations:
- Instant show/hide

### Detection Flow

**Reference**: `plugins/typing.lua:29-59, 69-96`

1. Player types in chat
2. `ChatTextChanged` fires
3. Plugin determines chat type/command
4. Sends type to server
5. Server broadcasts to PVS
6. Clients render indicator

### Range Calculation

**Reference**: `plugins/typing.lua:123-130, 188-189`

- **Default Range**: Chat range config (280 units, squared)
- **Custom Range**: Can be specified by chat type
- **Alpha**: `(1 - (distance / range)) * 255 * animation`

## Performance

- **PVS Optimization**: Only networked to nearby players
- **Rate Limiting**: Max one update per 0.2 seconds
- **Distance Culling**: Not rendered beyond chat range
- **Animation**: Optional, can be disabled

## See Also

- [Chat System](../systems/chat.md) - Creating custom chat types
- [Commands System](../systems/commands.md) - Custom commands
- [Chatbox Plugin](chatbox.md) - Enhanced chat interface
- [Language System](../libraries/language.md) - Localization
- Source: `plugins/typing.lua`
