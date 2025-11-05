# Chat System

> **Reference**: `gamemode/core/libs/sh_chatbox.lua`

The Helix chat system provides customizable chat types with different ranges, colors, and behaviors.

## ⚠️ Important: Use Built-in Chat System

**Always use `ix.chat.Register()`** for chat types. Don't create custom chat systems. Framework handles:
- Message formatting and display
- Range checking (who can hear)
- Color customization
- Typing indicators
- Dead player restrictions
- OOC cooldowns

## Core Chat Types

### Built-in Chat Classes

**Reference**: `gamemode/core/libs/sh_chatbox.lua:22`

- **ic** - In-character speech (local range)
- **ooc** - Out-of-character (global, everyone sees)
- **looc** - Local OOC (nearby players)
- **me** - Roleplay actions
- **it** - Environmental descriptions
- **w** - Whisper (short range)
- **y** - Yell (long range)
- **roll** - Dice roll
- **event** - Event messages (admin only)

## Registering Chat Types

### Use ix.chat.Register()

**Reference**: `gamemode/core/libs/sh_chatbox.lua:112`

```lua
-- Register in InitializedChatClasses hook
function PLUGIN:InitializedChatClasses()
    ix.chat.Register("radio", {
        format = "[RADIO] %s: \"%s\"",
        color = Color(150, 255, 150),
        CanHear = function(self, speaker, listener)
            -- Both must have radio item
            local speakerInv = speaker:GetCharacter():GetInventory()
            local listenerInv = listener:GetCharacter():GetInventory()

            local hasRadio = false
            for _, item in pairs(listenerInv:GetItems()) do
                if item.uniqueID == "item_radio" then
                    hasRadio = true
                    break
                end
            end

            return hasRadio
        end,
        prefix = {"/r", "/radio"},
        description = "Broadcast on radio"
    })
end
```

## Chat Properties

### Required Properties

```lua
{
    prefix = "/command",  -- or {"/cmd1", "/cmd2"}
}
```

### Optional Properties

```lua
{
    format = "%s says \"%s\"",      -- Message format
    color = Color(255, 255, 255),   -- Text color
    CanHear = 280,                  -- Hearing range (or function)
    description = "Description",    -- Help text
    indicator = "chatTyping",       -- Typing indicator text
    deadCanChat = false,            -- Allow dead players
    bNoIndicator = false,           -- Hide typing indicator
    font = "ixChatFont",            -- Custom font
    noSpaceAfter = false            -- Allow no space after prefix
}
```

## Hearing Range

### Number Range

```lua
CanHear = 280  -- Players within 280 units can hear
```

### Function Range

```lua
CanHear = function(self, speaker, listener)
    -- Custom logic
    return speaker:GetPos():Distance(listener:GetPos()) < 500
end
```

### Global (Everyone Hears)

```lua
CanHear = function(self, speaker, listener)
    return true
end
```

## Chat Commands

Use command system, not chat prefixes, for complex commands. See [Command System](commands.md).

## See Also

- [Command System](commands.md)
- Chat System: `gamemode/core/libs/sh_chatbox.lua`
