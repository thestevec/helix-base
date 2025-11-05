# Recognition Plugin

> **Reference**: `plugins/recognition.lua`

The Recognition plugin implements a character recognition system where players must "recognize" other characters before seeing their real names. Unrecognized characters appear as physical descriptions, enhancing immersion and roleplay.

## ⚠️ Important: Use Built-in Recognition

**Always use Helix's recognition system**. The framework provides:
- Automatic name hiding for unrecognized characters
- Multiple recognition methods (look, whisper, talk, yell)
- Faction-based global recognition
- Integration with chat, scoreboard, and descriptions

## Core Concepts

### Recognition System

- By default, you don't know other characters' names
- Unrecognized characters show as their physical description
- Once recognized, you always know their name
- Recognition is one-way (A recognizing B doesn't mean B recognizes A)

## Recognizing Characters

### Using Recognition Menu

Press F4 (ShowSpare1 key) to open recognition menu:
- **Looking At**: Recognize character you're aiming at
- **Whisper Range**: Recognize characters in whisper range
- **Talk Range**: Recognize characters in talk range
- **Yell Range**: Recognize characters in yell range

### Automatic Recognition

**Globally Recognized Factions**:
```lua
FACTION.isGloballyRecognized = true
```

Characters in these factions are automatically recognized by everyone (e.g., police, government officials).

## Character Functions

### character:Recognize(id)

**Server-side only**

```lua
-- Manually recognize a character
local success = character:Recognize(otherCharID)
```

### character:DoesRecognize(id)

**Shared**

```lua
-- Check if character recognizes another
if character:DoesRecognize(otherCharID) then
    -- They recognize them
end
```

## Display Behavior

### Unrecognized Characters

- Chat: Shows `[Physical description...]`
- Scoreboard: Shows "Unknown" (if enabled in config)
- Description: Shows "No recognition"

### Recognized Characters

- Chat: Shows actual character name
- Scoreboard: Shows actual character name
- Description: Shows character information

## Chat Types

**Recognized chat types** (show description instead of "Unknown"):
- `ic` - In-character
- `y` - Yell
- `w` - Whisper
- `me` - Actions

## Developer Hooks

### IsCharacterRecognized

```lua
function PLUGIN:IsCharacterRecognized(character, otherCharID)
    -- Return true to force recognition
    -- Return false/nil for normal behavior

    -- Example: Auto-recognize same faction
    local other = ix.char.loaded[otherCharID]
    if character:GetFaction() == other:GetFaction() then
        return true
    end
end
```

### IsPlayerRecognized

```lua
function PLUGIN:IsPlayerRecognized(client)
    -- Return true to force showing real name
    -- Return false/nil for normal behavior
end
```

## Complete Examples

### Example 1: Faction Auto-Recognition

```lua
-- Same faction members auto-recognize
function SCHEMA:IsCharacterRecognized(char, id)
    local other = ix.char.loaded[id]
    if other and char:GetFaction() == other:GetFaction() then
        return true
    end
end
```

### Example 2: Recognition Event

```lua
-- Reward for recognizing someone
function PLUGIN:CharacterRecognized(client, charID)
    client:Notify("You've recognized a new character!")
    -- Give XP or achievement
end
```

## Best Practices

### ✅ DO

- Roleplay the recognition process
- Use appropriate range for recognition
- Set factions as globally recognized when appropriate

### ❌ DON'T

- Don't metagame names
- Don't bypass recognition system
- Don't recognize everyone immediately

## Configuration

**scoreboardRecognition**: Controls if scoreboard respects recognition

## See Also

- [Character System](../systems/characters.md) - Character management
- [Factions System](../systems/factions.md) - Creating factions
- [Chat System](../systems/chat.md) - Chat integration
- Source: `plugins/recognition.lua`
