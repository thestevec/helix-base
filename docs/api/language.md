# Language API (ix.lang)

> **Reference**: `gamemode/core/libs/sh_language.lua`

The language API provides multi-language support for phrases throughout the framework. Languages are automatically loaded and phrases can be accessed with the global `L()` function.

## ⚠️ Important: Use Built-in Helix Language System

**Always use Helix's built-in localization** rather than hardcoding English strings. The framework automatically provides:
- Multiple language support
- Automatic loading from `languages/` directories
- Client preference system
- Server-side client-specific translation
- String formatting support
- Easy phrase management

## Core Concepts

### What is Localization?

The language system translates phrases:
- Define phrases in `languages/sh_languagename.lua`
- Access with `L("phraseKey")`
- Supports string formatting
- Client chooses their language
- Server translates per-client

### Key Terms

**Phrase**: A translatable string identified by key
**Language**: A collection of phrases (english, french, etc.)
**L()**: Global function to get translated phrases
**L2()**: Returns nil if phrase doesn't exist (vs falling back)

## Library Tables

### ix.lang.stored

**Reference**: `gamemode/core/libs/sh_language.lua:33`

**Realm**: Shared

```lua
-- Access language phrases
local english = ix.lang.stored["english"]
print(english["cmdRoll"])  -- "has rolled %d out of %d."

-- Check if phrase exists
if ix.lang.stored["french"] then
    print("French language loaded")
end
```

### ix.lang.names

**Reference**: `gamemode/core/libs/sh_language.lua:34`

**Realm**: Shared

Display names for languages.

```lua
-- Get language display name
print(ix.lang.names["english"])  -- "English"
print(ix.lang.names["french"])   -- "Français"
```

## Library Functions

### ix.lang.AddTable

**Reference**: `gamemode/core/libs/sh_language.lua:66`

**Realm**: Shared

```lua
ix.lang.AddTable(language, data)
```

Adds phrases to a language.

**Parameters**:
- `language` (string) - Language ID ("english", "french", etc.)
- `data` (table) - Table of phrase keys and translations

**Example**:
```lua
-- Add phrases in single-file plugin
ix.lang.AddTable("english", {
    myPluginTitle = "My Plugin",
    myPluginDesc = "Does something cool",
    playerGreeting = "Welcome, %s!"
})

ix.lang.AddTable("french", {
    myPluginTitle = "Mon Plugin",
    myPluginDesc = "Fait quelque chose de cool",
    playerGreeting = "Bienvenue, %s!"
})
```

### L (Global Function)

**Reference**: `gamemode/core/libs/sh_language.lua:73` (server), `:92` (client)

**Realm**: Shared

```lua
local phrase = L(key, client, ...)
```

Gets translated phrase for client's language.

**Parameters**:
- `key` (string) - Phrase key
- `client` (Player, server only) - Player to translate for
- `...` - Format arguments (for string.format)

**Returns**: (string) Translated phrase

**Example - Client**:
```lua
-- Simple phrase
print(L("cmdRoll"))  -- "has rolled %d out of %d."

-- With formatting
local msg = L("playerGreeting", client:Name())
chat.AddText(msg)  -- "Welcome, John!"

-- In UI
local label = vgui.Create("DLabel")
label:SetText(L("myPhrase"))
```

**Example - Server**:
```lua
-- Translate for specific client
local msg = L("welcome", client, client:Name())
client:ChatPrint(msg)

-- Notify with translation
client:NotifyLocalized("charCreated")  -- Uses L internally

-- Command feedback
return L("cmdSuccess", client)
```

### L2 (Global Function)

**Reference**: `gamemode/core/libs/sh_language.lua:82` (server), `:100` (client)

**Realm**: Shared

```lua
local phrase = L2(key, client, ...)
```

Like `L()` but returns `nil` if phrase doesn't exist instead of falling back.

**Parameters**: Same as `L()`

**Returns**: (string or nil) Translated phrase or nil

**Example**:
```lua
local optional = L2("optionalPhrase", client)

if optional then
    client:ChatPrint(optional)
else
    -- Phrase doesn't exist in their language
    client:ChatPrint("Default text")
end
```

## Defining Languages

### Language File Structure

**File Location**: `schema/languages/sh_languagename.lua` or `plugins/yourplugin/languages/sh_english.lua`

```lua
-- languages/sh_english.lua
NAME = "English"

LANGUAGE = {
    -- Command phrases
    cmdRoll = "has rolled %d out of %d.",
    cmdMe = "** %s %s",

    -- UI phrases
    char = "Character",
    charCreate = "Create Character",
    charDelete = "Delete Character",

    -- Notifications
    charCreated = "Character created!",
    charDeleted = "Character deleted.",

    -- Errors
    noCharSelected = "No character selected",
    invalidInput = "Invalid input: %s",

    -- Format strings
    welcomeMsg = "Welcome to the server, %s!",
    moneyReceived = "You received %s"
}
```

```lua
-- languages/sh_french.lua
NAME = "Français"

LANGUAGE = {
    cmdRoll = "a obtenu %d sur %d.",
    cmdMe = "** %s %s",

    char = "Personnage",
    charCreate = "Créer un Personnage",
    charDelete = "Supprimer le Personnage",

    charCreated = "Personnage créé !",
    charDeleted = "Personnage supprimé.",

    noCharSelected = "Aucun personnage sélectionné",
    invalidInput = "Entrée invalide : %s",

    welcomeMsg = "Bienvenue sur le serveur, %s !",
    moneyReceived = "Vous avez reçu %s"
}
```

## Complete Examples

### Plugin with Localization

```lua
PLUGIN.name = "Vendor System"
PLUGIN.uniqueID = "vendor"

-- Add English phrases
ix.lang.AddTable("english", {
    vendorMenuTitle = "Vendor",
    vendorBuyPrompt = "Buy %s for %s?",
    vendorBought = "Purchased %s",
    vendorCannotAfford = "You cannot afford this item",
    vendorOutOfStock = "This item is out of stock"
})

-- Add French phrases
ix.lang.AddTable("french", {
    vendorMenuTitle = "Vendeur",
    vendorBuyPrompt = "Acheter %s pour %s ?",
    vendorBought = "%s acheté",
    vendorCannotAfford = "Vous ne pouvez pas vous permettre cet article",
    vendorOutOfStock = "Cet article est en rupture de stock"
})

-- Use in code
function PLUGIN:ShowVendorMenu(client)
    net.Start("VendorMenu")
        net.WriteString(L("vendorMenuTitle", client))
    net.Send(client)
end
```

### Command with Localization

```lua
ix.command.Add("Greet", {
    description = "Greet another player",
    arguments = ix.type.player,
    OnRun = function(self, client, target)
        -- Returns localized string to command executor
        return L("greetSuccess", client, target:Name())
    end
})

-- In languages file
LANGUAGE = {
    greetSuccess = "You greeted %s"
}
```

### Fallback Translations

```lua
-- Try localized, fallback to English, then key
local function GetPhrase(key, client, ...)
    local phrase = L2(key, client, ...)

    if not phrase then
        -- Try English fallback
        local english = ix.lang.stored.english
        phrase = english and english[key]

        if phrase then
            phrase = string.format(phrase, ...)
        else
            -- Last resort: use key
            phrase = key
        end
    end

    return phrase
end
```

### UI with Language Selector

```lua
-- Client-side language dropdown
local languages = {}
for id, name in pairs(ix.lang.names) do
    table.insert(languages, {id = id, name = name})
end

local dropdown = vgui.Create("DComboBox")
dropdown:SetValue(ix.option.Get("language", "english"))

for _, lang in ipairs(languages) do
    dropdown:AddChoice(lang.name, lang.id)
end

dropdown.OnSelect = function(_, index, text, id)
    ix.option.Set("language", id)
    -- Refresh UI with new language
end
```

## Best Practices

### ✅ DO

- Use `L()` for all user-facing strings
- Provide translations for supported languages
- Use descriptive phrase keys (e.g., "charCreate" not "cc")
- Use format strings (%s, %d) for dynamic content
- Test with multiple languages
- Group related phrases together
- Keep phrase keys consistent across languages

### ❌ DON'T

- Don't hardcode English strings
- Don't use generic keys like "msg1", "text2"
- Don't forget format arguments
- Don't duplicate phrases (reuse existing)
- Don't use special characters in keys
- Don't modify framework language files
- Don't assume everyone uses English

## Common Patterns

### Conditional Phrase

```lua
local statusKey = isOnline and "playerOnline" or "playerOffline"
client:ChatPrint(L(statusKey, client))
```

### Phrase with Multiple Arguments

```lua
LANGUAGE = {
    tradeOffer = "%s offered to trade %s for %s"
}

-- Usage
local msg = L("tradeOffer", client, sender:Name(), itemA, itemB)
```

### Error Messages

```lua
if not character then
    return L("noCharSelected", client)
end

if amount <= 0 then
    return L("invalidAmount", client, amount)
end
```

## Common Issues

### Phrase Returns Key Instead of Translation

**Cause**: Phrase not defined in current language.

**Fix**: Add phrase to language file or use fallback:
```lua
-- Add to language file
LANGUAGE = {
    myPhrase = "My Translation"
}
```

### Format Arguments Not Working

**Cause**: Missing arguments to `L()`.

**Fix**: Provide all format arguments:
```lua
-- WRONG
L("welcomeMsg", client)  -- Missing name argument

-- CORRECT
L("welcomeMsg", client, client:Name())
```

### Language Not Loading

**Cause**: Wrong file location or name.

**Fix**: Use correct naming:
```
plugins/myplugin/languages/sh_english.lua  ✓
plugins/myplugin/language/english.lua      ✗
plugins/myplugin/languages/en.lua          ✗
```

## Related Hooks

No specific hooks, but integrate with options:

```lua
-- Player language preference
local lang = ix.option.Get(client, "language", "english")
```

## See Also

- [Option API](option.md) - Client language preference
- [Notice API](notice.md) - Localized notifications
- [Command API](command.md) - Command descriptions can use "@phraseKey"
- Source: `gamemode/core/libs/sh_language.lua`
