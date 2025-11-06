# Localization System (ix.lang)

> **Reference**: `gamemode/core/libs/sh_language.lua`

The localization system provides multi-language support for displaying text in players' preferred languages. It automatically loads language files and provides global functions for translating phrases.

## ⚠️ Important: Use Built-in Localization

**Always use Helix's localization system** for user-facing text rather than hardcoding English strings. The framework provides:
- Automatic language file loading from schema and plugins
- Player preference-based translation
- String formatting support with arguments
- Fallback to English for missing phrases
- Global `L()` and `L2()` functions for easy translation
- Server-side client-specific localization

## Core Concepts

### What is Localization?

Localization (L10n) allows displaying text in different languages based on player preferences. Instead of hardcoding text like `"Hello, Player!"`, you use phrase keys like `"greeting"` that translate to different languages.

**Benefits**:
- Support international player base
- Easy text updates without code changes
- Consistent terminology across gamemode
- Professional multi-language support

### Key Terms

- **Phrase**: A text string identified by a unique key (e.g., `"charCreated"`)
- **Language File**: Lua file containing phrase translations for one language
- **Language ID**: Language identifier (e.g., `"english"`, `"french"`, `"german"`)
- **L()**: Global function for translating phrases
- **L2()**: Global function that returns `nil` if phrase doesn't exist

### Language File Structure

**Location**:
- Schema: `schema/languages/sh_languageid.lua`
- Plugin: `plugins/myplugin/languages/sh_languageid.lua`

**Format**:
```lua
NAME = "English"  -- Display name (optional)

LANGUAGE = {
    phraseKey = "Phrase Text",
    anotherPhrase = "Another phrase with %s formatting",
    thirdPhrase = "Phrase with multiple %s and %d arguments"
}
```

## Using Localization

### L() Function

**Reference**: `gamemode/core/libs/sh_language.lua:73` (server), `92` (client)

```lua
local text = L(key, ...)
```

Translates a phrase key to localized text. On server, pass client as second argument.

**Client-Side**:
```lua
local text = L(phraseKey, arg1, arg2, ...)
```

**Server-Side**:
```lua
local text = L(phraseKey, client, arg1, arg2, ...)
```

**Complete Example**:
```lua
-- CLIENT: Simple translation
local greeting = L("greeting")  -- "Hello!"

-- CLIENT: Translation with formatting
local welcome = L("welcomeMsg", "John")  -- "Welcome, John!"

-- CLIENT: Multiple arguments
local info = L("itemInfo", "Medkit", "Restores health")
    -- "Name: Medkit\nDescription: Restores health"

-- SERVER: Client-specific translation
function PLUGIN:ShowGreeting(client)
    local text = L("greeting", client)
    client:ChatPrint(text)  -- Shows in client's language
end

-- SERVER: With formatting
function PLUGIN:NotifyPurchase(client, itemName, cost)
    local text = L("itemPurchased", client, itemName, cost)
    client:Notify(text)  -- "You purchased %s for %d tokens"
end

-- In UI panels
function PANEL:Paint(w, h)
    draw.SimpleText(L("inventory"), "MyFont", 10, 10, color_white)
end

-- Button labels
function PANEL:Init()
    self.button = self:Add("DButton")
    self.button:SetText(L("save"))
    self.button:SetSize(100, 30)
end

-- Chat messages
hook.Add("OnPlayerChat", "TranslateChat", function(client, text)
    if text == "/time" then
        client:ChatPrint(L("currentTime", client, os.date("%H:%M")))
        return true
    end
end)
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't hardcode English text
client:Notify("You have created your character.")  -- Not localized!

-- WRONG: Don't concatenate translations
local text = L("hello") .. " " .. L("player")  -- Create single phrase instead

-- WRONG: Don't forget client parameter on server
local text = L("greeting", "argument")  -- On server, second param is client!
```

### L2() Function

**Reference**: `gamemode/core/libs/sh_language.lua:82` (server), `100` (client)

```lua
local text = L2(key, ...)
```

Like `L()`, but returns `nil` if phrase doesn't exist instead of returning the key.

**Complete Example**:
```lua
-- CLIENT: Check if phrase exists
local description = L2("itemDescription", itemID)

if description then
    panel:SetText(description)
else
    panel:SetText(L("noDesc"))  -- "No description available"
end

-- SERVER: Conditional translation
function PLUGIN:ShowOptionalMessage(client, messageKey)
    local text = L2(messageKey, client)

    if text then
        client:ChatPrint(text)
    else
        print("Warning: Missing translation for " .. messageKey)
    end
end

-- Fallback to default
function PLUGIN:GetItemName(item)
    return L2("item_" .. item.uniqueID) or item.name or L("unknown")
end

-- Check before displaying
function PANEL:SetTitle(phraseKey)
    local translated = L2(phraseKey)

    if translated then
        self.title:SetText(translated)
        self.title:SizeToContents()
    end
end
```

### ix.lang.AddTable

**Reference**: `gamemode/core/libs/sh_language.lua:66`

```lua
ix.lang.AddTable(language, phraseTable)
```

Adds phrases to a language. Useful for single-file plugins without separate language files.

**Complete Example**:
```lua
-- In single-file plugin
PLUGIN.name = "Simple Plugin"

if (CLIENT) then
    ix.lang.AddTable("english", {
        simplePluginTitle = "Simple Plugin",
        simplePluginDesc = "This is a simple plugin",
        simplePluginButton = "Click Me"
    })

    ix.lang.AddTable("french", {
        simplePluginTitle = "Plugin Simple",
        simplePluginDesc = "Ceci est un plugin simple",
        simplePluginButton = "Cliquez-moi"
    })

    ix.lang.AddTable("german", {
        simplePluginTitle = "Einfaches Plugin",
        simplePluginDesc = "Dies ist ein einfaches Plugin",
        simplePluginButton = "Klick mich"
    })
end

-- Add phrases dynamically
function PLUGIN:InitializedPlugins()
    ix.lang.AddTable("english", {
        customGreeting = "Hello, %s! Welcome to the server.",
        customFarewell = "Goodbye, %s! Come back soon."
    })
end

-- Add item-specific phrases
function PLUGIN:InitializedItems()
    for uniqueID, itemTable in pairs(ix.item.list) do
        ix.lang.AddTable("english", {
            ["item_" .. uniqueID] = itemTable.name,
            ["itemDesc_" .. uniqueID] = itemTable.description
        })
    end
end

-- Programmatically generate phrases
function PLUGIN:GeneratePhrases()
    local weapons = {"pistol", "rifle", "shotgun"}
    local phrases = {}

    for _, weapon in ipairs(weapons) do
        phrases["weapon_" .. weapon] = weapon:upper()
        phrases["weaponDesc_" .. weapon] = "A " .. weapon .. " weapon"
    end

    ix.lang.AddTable("english", phrases)
end
```

## Creating Language Files

### Plugin Language File

**File**: `plugins/myplugin/languages/sh_english.lua`

```lua
NAME = "English"

LANGUAGE = {
    -- Plugin title and description
    myPluginTitle = "My Plugin",
    myPluginDesc = "This plugin does something cool",

    -- Commands
    cmdDoSomething = "Performs an action.",
    cmdDoSomethingUsage = "Usage: /dosomething <target>",

    -- Notifications
    actionSuccess = "Action completed successfully!",
    actionFailed = "Action failed: %s",

    -- UI elements
    buttonSave = "Save",
    buttonCancel = "Cancel",
    buttonApply = "Apply",

    -- Status messages
    statusProcessing = "Processing...",
    statusComplete = "Complete!",

    -- Errors
    errorInvalidTarget = "Invalid target specified!",
    errorNoPermission = "You don't have permission!",

    -- Item names
    itemMedkit = "Medical Kit",
    itemMedkitDesc = "Restores %d health points",

    -- Character info
    charLevel = "Level: %d",
    charExperience = "Experience: %d/%d",

    -- Formatting examples
    playerJoined = "%s has joined the game.",
    playerLeft = "%s has left the game.",
    moneyReceived = "You received %s.",
    itemPurchased = "You purchased %s for %s.",
}
```

### Schema Language File

**File**: `schema/languages/sh_english.lua`

```lua
NAME = "English"

LANGUAGE = {
    -- Schema-specific phrases
    schemaWelcome = "Welcome to our roleplay server!",
    schemaRules = "Please read the rules at F1 menu.",

    -- Faction names
    factionCitizen = "Citizen",
    factionCitizenDesc = "A regular citizen of City 17",
    factionPolice = "Metro Police",
    factionPoliceDesc = "Law enforcement officer",

    -- Class names
    classWorker = "Worker",
    classWorkerDesc = "Works at the factory",

    -- Custom systems
    loyaltyPoints = "Loyalty Points: %d",
    rationReceived = "You received your ration.",

    -- Events
    eventRaidStarting = "A raid is starting in %d minutes!",
    eventMarketOpen = "The market is now open.",
}
```

### Multiple Language Support

**File**: `plugins/myplugin/languages/sh_french.lua`

```lua
NAME = "Français"

LANGUAGE = {
    myPluginTitle = "Mon Plugin",
    myPluginDesc = "Ce plugin fait quelque chose de cool",

    cmdDoSomething = "Effectue une action.",
    cmdDoSomethingUsage = "Utilisation: /dosomething <cible>",

    actionSuccess = "Action terminée avec succès!",
    actionFailed = "L'action a échoué: %s",

    buttonSave = "Sauvegarder",
    buttonCancel = "Annuler",
    buttonApply = "Appliquer",

    errorInvalidTarget = "Cible invalide spécifiée!",
    errorNoPermission = "Vous n'avez pas la permission!",

    playerJoined = "%s a rejoint le jeu.",
    playerLeft = "%s a quitté le jeu.",
}
```

## Complete Example

### Multi-Language Shop System

```lua
PLUGIN.name = "Shop System"
PLUGIN.description = "A simple shop with localization"

-- English translations
ix.lang.AddTable("english", {
    shopTitle = "Shop",
    shopWelcome = "Welcome to the shop!",
    shopBuy = "Buy",
    shopSell = "Sell",
    shopClose = "Close",
    shopItemCost = "Cost: %s",
    shopPurchaseSuccess = "You purchased %s for %s!",
    shopPurchaseFailed = "You don't have enough money!",
    shopSellSuccess = "You sold %s for %s!",
    shopNoItems = "You have no items to sell.",
    shopConfirmPurchase = "Purchase %s for %s?",
    shopConfirmSell = "Sell %s for %s?",
})

-- French translations
ix.lang.AddTable("french", {
    shopTitle = "Boutique",
    shopWelcome = "Bienvenue à la boutique!",
    shopBuy = "Acheter",
    shopSell = "Vendre",
    shopClose = "Fermer",
    shopItemCost = "Coût: %s",
    shopPurchaseSuccess = "Vous avez acheté %s pour %s!",
    shopPurchaseFailed = "Vous n'avez pas assez d'argent!",
    shopSellSuccess = "Vous avez vendu %s pour %s!",
    shopNoItems = "Vous n'avez aucun article à vendre.",
    shopConfirmPurchase = "Acheter %s pour %s?",
    shopConfirmSell = "Vendre %s pour %s?",
})

if (SERVER) then
    function PLUGIN:PurchaseItem(client, itemID, cost)
        local char = client:GetCharacter()
        local money = char:GetMoney()

        if money >= cost then
            char:TakeMoney(cost)
            char:GetInventory():Add(itemID)

            local itemName = ix.item.list[itemID].name
            local costStr = ix.currency.Get(cost)
            client:Notify(L("shopPurchaseSuccess", client, itemName, costStr))
        else
            client:Notify(L("shopPurchaseFailed", client))
        end
    end
end

if (CLIENT) then
    local PANEL = {}

    function PANEL:Init()
        self:SetTitle(L("shopTitle"))
        self:SetSize(600, 400)
        self:Center()

        -- Welcome text
        self.welcome = self:Add("DLabel")
        self.welcome:SetText(L("shopWelcome"))
        self.welcome:Dock(TOP)

        -- Buy button
        self.buyButton = self:Add("DButton")
        self.buyButton:SetText(L("shopBuy"))
        self.buyButton:Dock(TOP)

        -- Sell button
        self.sellButton = self:Add("DButton")
        self.sellButton:SetText(L("shopSell"))
        self.sellButton:Dock(TOP)

        -- Close button
        self.closeButton = self:Add("DButton")
        self.closeButton:SetText(L("shopClose"))
        self.closeButton:Dock(BOTTOM)
        self.closeButton.DoClick = function()
            self:Close()
        end
    end

    function PANEL:ShowPurchaseConfirm(itemName, cost)
        local costStr = ix.currency.Get(cost)
        local message = L("shopConfirmPurchase", itemName, costStr)

        Derma_Query(
            message,
            L("shopTitle"),
            L("yes"),
            function()
                self:ConfirmPurchase(itemID, cost)
            end,
            L("no")
        )
    end

    vgui.Register("ixShop", PANEL, "DFrame")
end
```

## Best Practices

### ✅ DO

- Use `L()` for all user-facing text
- Create separate language files in `languages/` folder
- Use descriptive phrase keys (e.g., `"charCreated"` not `"msg1"`)
- Include formatting arguments in phrases
- Provide English translations at minimum
- Use `L2()` when you need to check if phrase exists
- Group related phrases with common prefixes
- Document format arguments in comments

### ❌ DON'T

- Don't hardcode English text in code
- Don't use the same phrase key for different meanings
- Don't create overly generic phrase keys (e.g., `"text"`, `"message"`)
- Don't forget format arguments when calling `L()`
- Don't modify core language files directly
- Don't assume English is the only language
- Don't create separate phrases for minor variations—use formatting
- Don't forget client parameter when using `L()` on server

## Common Patterns

### Pattern 1: Command Descriptions

```lua
-- In language file
LANGUAGE = {
    cmdGiveItem = "Gives an item to a player.",
    cmdGiveItemArg = "<string name> <string item>",
    cmdKick = "Kicks a player from the server.",
    cmdKickArg = "<string name> [string reason]",
}

-- In command definition
COMMAND.description = "@cmdGiveItem"
COMMAND.arguments = ix.type.string("name")
```

### Pattern 2: Dynamic Notifications

```lua
-- Language phrases
ix.lang.AddTable("english", {
    notifyItemGiven = "You gave %s to %s.",
    notifyItemReceived = "You received %s from %s.",
})

-- Usage
function PLUGIN:GiveItem(giver, receiver, item)
    giver:Notify(L("notifyItemGiven", giver, item:GetName(), receiver:Name()))
    receiver:Notify(L("notifyItemReceived", receiver, item:GetName(), giver:Name()))
end
```

### Pattern 3: Menu Options

```lua
-- Language file
LANGUAGE = {
    menuInventory = "Inventory",
    menuCharacter = "Character",
    menuSkills = "Skills",
    menuSettings = "Settings",
}

-- Menu creation
function PANEL:CreateTabs()
    self.tabs = self:Add("DPropertySheet")

    self.tabs:AddSheet(L("menuInventory"), inventoryPanel)
    self.tabs:AddSheet(L("menuCharacter"), characterPanel)
    self.tabs:AddSheet(L("menuSkills"), skillsPanel)
    self.tabs:AddSheet(L("menuSettings"), settingsPanel)
end
```

### Pattern 4: Formatted Messages

```lua
-- Language file
LANGUAGE = {
    combatLog = "%s %s %s with %s for %d damage.",
    combatKilled = "killed",
    combatHit = "hit",
}

-- Usage
function PLUGIN:LogCombat(attacker, victim, weapon, damage, wasKilled)
    local action = wasKilled and L("combatKilled") or L("combatHit")
    local message = L("combatLog",
        attacker:Name(),
        action,
        victim:Name(),
        weapon:GetClass(),
        damage
    )

    print(message)
end
```

## Common Issues

### Issue: Phrase Returns Key Instead of Text

**Cause**: Phrase not defined or typo in phrase key
**Fix**: Check spelling and ensure phrase exists in language file

```lua
-- Check if phrase exists
local text = L2("myPharse")  -- Typo: "myPharse" instead of "myPhrase"

if not text then
    print("Missing phrase: myPharse")
    text = L("noDesc")  -- Use fallback
end
```

### Issue: Formatting Arguments Wrong

**Cause**: Wrong number or type of arguments
**Fix**: Match format specifiers with arguments

```lua
-- WRONG
LANGUAGE = {
    message = "Player %s has %d items"  -- Expects string and number
}
L("message", 5, "John")  -- Wrong order!

-- CORRECT
L("message", "John", 5)  -- Correct order
```

### Issue: Server-Side Translation Not Working

**Cause**: Forgot client parameter
**Fix**: Always pass client as second argument on server

```lua
-- WRONG (on server)
local text = L("greeting", "argument")

-- CORRECT (on server)
local text = L("greeting", client, "argument")
```

## See Also

- [Configuration System](../systems/configuration.md) - Language preference setting
- [Commands System](../systems/commands.md) - Localized command descriptions
- [Plugin System](../plugins/plugin-system.md) - Adding plugin languages
- Source: `gamemode/core/libs/sh_language.lua`
- Example: `plugins/area/languages/sh_english.lua`
- Example: `gamemode/languages/sh_english.lua`
