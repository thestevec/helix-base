# Util Library (ix.util)

> **Reference**: `gamemode/core/sh_util.lua`

The util library provides essential helper functions for file inclusion, type checking, string manipulation, color operations, player finding, and various other utilities used throughout the Helix framework.

## ⚠️ Important: Use Built-in Helix Functions

**Always use Helix's ix.util functions** rather than creating custom implementations. The framework provides:
- Automatic realm detection and file sending for Include/IncludeDir
- Type-safe validation and sanitization
- Optimized cached operations (materials, etc.)
- Cross-realm compatible utilities
- Integration with Helix type system

## Core Concepts

### What is ix.util?

The util library is the foundation of Helix's helper functions. It handles critical operations like:
- **File inclusion** with automatic realm management (sv_, cl_, sh_ prefixes)
- **Type system** integration for validating and sanitizing data
- **Player utilities** for finding and iterating over players/characters
- **String utilities** for formatting and matching
- **Client-side rendering** helpers for UI effects

### Realm Prefixes

Helix uses file prefixes to determine where code runs:
- `sh_` - **Shared** - Runs on both server and client
- `sv_` - **Server** - Runs only on server
- `cl_` - **Client** - Runs only on client

## File Inclusion Functions

### ix.util.Include

**Reference**: `gamemode/core/sh_util.lua:39`

Includes a Lua file with automatic realm detection based on the filename prefix.

```lua
-- Include a shared file (runs on both server and client)
ix.util.Include("sh_myfile.lua")

-- Include a server file (only runs on server)
ix.util.Include("sv_database.lua")

-- Include a client file (sent to and runs on client only)
ix.util.Include("cl_hud.lua")

-- Manual realm override (rarely needed)
ix.util.Include("myfile.lua", "client")
```

**How it works:**
- **Server files** (`sv_`): Included only on server
- **Shared files** (`sh_`): AddCSLuaFile called on server, included on both
- **Client files** (`cl_`): AddCSLuaFile called on server, included only on client

**⚠️ Do NOT**:
```lua
-- WRONG: Don't use include() directly for shared files
include("sh_myfile.lua")  -- Won't send to clients!

-- WRONG: Don't manually call AddCSLuaFile
AddCSLuaFile("cl_hud.lua")
include("cl_hud.lua")  -- Use ix.util.Include instead!
```

### ix.util.IncludeDir

**Reference**: `gamemode/core/sh_util.lua:71`

Includes all Lua files in a directory, automatically determining realm from prefixes.

```lua
-- Include all files from a directory
ix.util.IncludeDir("core/libs")

-- From the gamemode initialization (gamemode/shared.lua:82-86)
ix.util.IncludeDir("core/libs/thirdparty")
ix.util.IncludeDir("core/libs")
ix.util.IncludeDir("core/derma")
ix.util.IncludeDir("core/hooks")

-- Include from lua/ folder directly
ix.util.IncludeDir("mymodule/files", true)
```

**Parameters:**
- `directory` - Directory path relative to gamemode/ or schema/
- `bFromLua` - Set to `true` to search from lua/ folder instead

**⚠️ Do NOT**:
```lua
-- WRONG: Don't manually loop and include files
for _, file in ipairs(file.Find("core/libs/*.lua", "LUA")) do
    include("core/libs/" .. file)  -- Missing AddCSLuaFile, realm detection!
end
```

## Type System Functions

### ix.util.SanitizeType

**Reference**: `gamemode/core/sh_util.lua:135`

Sanitizes input to ensure valid type, returning safe defaults if invalid.

```lua
-- Convert string to number safely
local health = ix.util.SanitizeType(ix.type.number, "100")
print(health)  -- 100

-- Invalid input returns default
local invalid = ix.util.SanitizeType(ix.type.number, "abc")
print(invalid)  -- 0

-- Boolean conversion
local enabled = ix.util.SanitizeType(ix.type.bool, 1)
print(enabled)  -- true

-- Color validation
local color = ix.util.SanitizeType(ix.type.color, {r = 255, g = 0, b = 0})
print(color)  -- Color(255, 0, 0, 255)
```

**Supported types** (see `ix.type` at line 5):
- `ix.type.string` - String values
- `ix.type.number` - Numeric values
- `ix.type.bool` - Boolean values
- `ix.type.color` - Color tables
- `ix.type.vector` - Vector values
- `ix.type.array` - Table arrays

### ix.util.GetTypeFromValue

**Reference**: `gamemode/core/sh_util.lua:187`

Returns the `ix.type` constant for a given value.

```lua
local typeId = ix.util.GetTypeFromValue("hello")
print(typeId)  -- 2 (ix.type.string)

local typeId = ix.util.GetTypeFromValue(123)
print(typeId)  -- 8 (ix.type.number)

local character = client:GetCharacter()
local typeId = ix.util.GetTypeFromValue(character)
print(typeId)  -- 64 (ix.type.character)
```

### ix.util.IsColor

**Reference**: `gamemode/core/sh_util.lua:106`

Checks if a value is a color table (necessary for non-metatable colors).

```lua
local color = Color(255, 0, 0)
if ix.util.IsColor(color) then
    print("Valid color!")
end

-- Check user input
local input = {r = 100, g = 150, b = 200, a = 255}
if ix.util.IsColor(input) then
    -- Safe to use as color
end
```

## Player Utilities

### ix.util.FindPlayer

**Reference**: `gamemode/core/sh_util.lua:234`

Finds a player by name or Steam ID with fuzzy matching.

```lua
-- Find by partial name
local player = ix.util.FindPlayer("John")  -- Finds "John Smith"

-- Find by Steam ID
local player = ix.util.FindPlayer("STEAM_0:1:23456789")

-- With Lua patterns (use carefully!)
local player = ix.util.FindPlayer("^Admin", true)

-- Use in commands
function COMMAND:OnRun(client, arguments)
    local target = ix.util.FindPlayer(arguments[1])

    if not target then
        return "@playerNotFound"
    end

    -- Do something with target
end
```

### ix.util.GetCharacters

**Reference**: `gamemode/core/sh_util.lua:375`

Iterator for all players with loaded characters.

```lua
-- Iterate over all characters
for client, character in ix.util.GetCharacters() do
    print(client:Name(), character:GetName())
end

-- Give money to all characters
for client, character in ix.util.GetCharacters() do
    character:GiveMoney(100)
    client:Notify("You received $100!")
end

-- Count total characters
local count = 0
for client, character in ix.util.GetCharacters() do
    count = count + 1
end
print("Active characters: " .. count)
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't iterate player.GetAll() and manually check characters
for _, client in ipairs(player.GetAll()) do
    local character = client:GetCharacter()
    if character then  -- Manual check!
        -- Use ix.util.GetCharacters() instead
    end
end
```

## String Utilities

### ix.util.StringMatches

**Reference**: `gamemode/core/sh_util.lua:258`

Fuzzy string matching (case-insensitive, substring matching).

```lua
-- Exact match
ix.util.StringMatches("Hello", "Hello")  -- true

-- Case insensitive
ix.util.StringMatches("Hello", "hello")  -- true

-- Substring match
ix.util.StringMatches("Hello World", "World")  -- true
ix.util.StringMatches("JohnSmith", "Smith")  -- true

-- Use in search
function FindItemByName(name)
    for _, item in pairs(ix.item.list) do
        if ix.util.StringMatches(item.name, name) then
            return item
        end
    end
end
```

### ix.util.FormatStringNamed

**Reference**: `gamemode/core/sh_util.lua:283`

Replaces named placeholders in strings with values.

```lua
-- Named arguments
local text = ix.util.FormatStringNamed(
    "Hello {name}, you have {money} credits!",
    {name = "John", money = 500}
)
print(text)  -- "Hello John, you have 500 credits!"

-- Ordered arguments
local text = ix.util.FormatStringNamed(
    "Hello {name}, you have {money} credits!",
    "John", 500
)
print(text)  -- "Hello John, you have 500 credits!"

-- Use in notifications
client:Notify(ix.util.FormatStringNamed(
    "@itemPurchased",
    {item = item.name, cost = item.price}
))
```

### ix.util.ExpandCamelCase

**Reference**: `gamemode/core/sh_util.lua:322`

Converts CamelCase to spaced words with special handling for acronyms.

```lua
print(ix.util.ExpandCamelCase("HelloWorld"))
-- > "Hello World"

print(ix.util.ExpandCamelCase("PlayerOOC"))
-- > "Player OOC"  -- OOC automatically uppercased

print(ix.util.ExpandCamelCase("ItemName"))
-- > "Item Name"

-- Keep first letter lowercase
print(ix.util.ExpandCamelCase("myVariable", true))
-- > "my Variable"
```

**Special uppercase words**: `ooc`, `looc`, `afk`, `url` (line 307-312)

## Color Utilities

### ix.util.DimColor

**Reference**: `gamemode/core/sh_util.lua:119`

Returns a dimmed version of a color.

```lua
local original = Color(100, 100, 100, 255)
local dimmed = ix.util.DimColor(original, 0.5)
print(dimmed)  -- Color(50, 50, 50, 255)

-- Custom alpha
local dimmed = ix.util.DimColor(original, 0.75, 128)
print(dimmed)  -- Color(75, 75, 75, 128)

-- Use in UI
function PANEL:Paint(w, h)
    local color = self:GetSkin().Colours.Header
    local dimColor = ix.util.DimColor(color, 0.5)

    surface.SetDrawColor(dimColor)
    surface.DrawRect(0, 0, w, h)
end
```

## Material Utilities

### ix.util.GetMaterial

**Reference**: `gamemode/core/sh_util.lua:221`

Returns cached material or creates and caches if doesn't exist.

```lua
-- Get cached material (CLIENT)
local icon = ix.util.GetMaterial("icon16/user.png")

-- Reuse without re-creating Material()
function PANEL:Paint(w, h)
    local mat = ix.util.GetMaterial("materials/my_background.png")
    surface.SetMaterial(mat)
    surface.DrawTexturedRect(0, 0, w, h)
end
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't create Material() every frame
function PANEL:Paint(w, h)
    local mat = Material("icon16/user.png")  -- Creates new material each frame!
    surface.SetMaterial(mat)
    surface.DrawTexturedRect(0, 0, w, h)
end

-- CORRECT: Use ix.util.GetMaterial for caching
function PANEL:Paint(w, h)
    local mat = ix.util.GetMaterial("icon16/user.png")  -- Cached!
    surface.SetMaterial(mat)
    surface.DrawTexturedRect(0, 0, w, h)
end
```

## Client-Side Rendering (CLIENT)

### ix.util.DrawBlur

**Reference**: `gamemode/core/sh_util.lua:395`

Draws blur effect on a panel (respects user's cheap blur setting).

```lua
-- In a VGUI panel
function PANEL:Paint(width, height)
    -- Draw blur background
    ix.util.DrawBlur(self)

    -- Custom intensity
    ix.util.DrawBlur(self, 8)  -- More intense blur

    -- Custom passes and alpha
    ix.util.DrawBlur(self, 5, 0.3, 200)
end

-- Full example
function PANEL:Init()
    self:SetSize(400, 300)
    self:Center()
end

function PANEL:Paint(w, h)
    ix.util.DrawBlur(self, 5)  -- Blur background

    -- Draw semi-transparent background
    surface.SetDrawColor(0, 0, 0, 150)
    surface.DrawRect(0, 0, w, h)

    -- Draw content...
end
```

### ix.util.DrawText

**Reference**: `gamemode/core/sh_util.lua:474`

Draws text with shadow (helper for draw.TextShadow).

```lua
-- Draw text with shadow (CLIENT)
hook.Add("HUDPaint", "MyHUD", function()
    ix.util.DrawText(
        "Health: 100",
        100, 100,
        Color(255, 255, 255),
        TEXT_ALIGN_LEFT,
        TEXT_ALIGN_TOP,
        "ixGenericFont"
    )
end)

-- Centered text
ix.util.DrawText(
    "Welcome!",
    ScrW() / 2, 50,
    color_white,
    TEXT_ALIGN_CENTER,
    TEXT_ALIGN_TOP,
    "ixTitleFont"
)
```

## Time Utilities

### ix.util.GetStringTime

**Reference**: `gamemode/core/sh_util.lua:974`

Parses time strings into seconds.

```lua
-- Time units: s, m, h, d, w, mo, y
local seconds = ix.util.GetStringTime("5m")
print(seconds)  -- 300 (5 minutes)

local seconds = ix.util.GetStringTime("2h30m")
print(seconds)  -- 9000 (2.5 hours)

local seconds = ix.util.GetStringTime("1d")
print(seconds)  -- 86400 (1 day)

-- Use in ban command
function COMMAND:OnRun(client, arguments)
    local target = ix.util.FindPlayer(arguments[1])
    local duration = ix.util.GetStringTime(arguments[2])

    -- Ban for parsed duration
    target:Ban(duration)
end

-- Complex time string
local seconds = ix.util.GetStringTime("5y2d7w")
print(seconds)  -- 162086400 (5 years, 2 days, 7 weeks)
```

**Supported units** (line 949-956):
- `s` - Seconds
- `m` - Minutes
- `h` - Hours
- `d` - Days
- `w` - Weeks
- `mo` - Months (30 days)
- `y` - Years (365 days)

### ix.util.GetUTCTime

**Reference**: `gamemode/core/sh_util.lua:940`

Gets the current time in UTC timezone.

```lua
local utcTime = ix.util.GetUTCTime()
print(utcTime)  -- Offset from local time to UTC

-- Use for synchronized timestamps
local timestamp = os.time() + ix.util.GetUTCTime()
```

## Table Utilities

### ix.util.MetatableSafeTableMerge

**Reference**: `gamemode/core/sh_util.lua:1140`

Merges tables without overwriting metatable objects.

```lua
-- Merge configuration
local defaults = {
    maxHealth = 100,
    armor = 50,
    items = {
        "item_medkit",
        "item_bandage"
    }
}

local custom = {
    maxHealth = 150,
    items = {
        "item_medkit",
        "item_food"
    }
}

ix.util.MetatableSafeTableMerge(defaults, custom)
-- defaults.maxHealth = 150
-- defaults.items = {"item_medkit", "item_food", "item_bandage"}
```

**⚠️ Do NOT**:
```lua
-- WRONG: table.Merge overwrites metatable objects
table.Merge(destination, source)  -- Can break metatable objects!

-- CORRECT: Use MetatableSafeTableMerge
ix.util.MetatableSafeTableMerge(destination, source)
```

## Complete Example

```lua
-- Plugin that loads custom items from a directory
PLUGIN.name = "Custom Items"
PLUGIN.description = "Loads custom items"
PLUGIN.author = "YourName"

function PLUGIN:InitializedPlugins()
    -- Include all item files from items directory
    ix.util.IncludeDir("plugins/" .. self.uniqueID .. "/items")

    print("Loaded custom items!")
end

-- Command to find and give item to player
ix.command.Add("GiveItemFuzzy", {
    description = "Give item by fuzzy name search",
    arguments = {
        ix.type.player,
        ix.type.text
    },
    OnRun = function(self, client, target, itemName)
        -- Find item using fuzzy string matching
        local foundItem = nil

        for uniqueID, itemData in pairs(ix.item.list) do
            if ix.util.StringMatches(itemData.name, itemName) then
                foundItem = uniqueID
                break
            end
        end

        if not foundItem then
            return "@itemNotFound"
        end

        -- Give item to target's inventory
        local character = target:GetCharacter()
        local inventory = character:GetInventory()

        inventory:Add(foundItem, 1)

        -- Notify with formatted string
        client:Notify(ix.util.FormatStringNamed(
            "Gave {item} to {player}",
            {item = ix.item.list[foundItem].name, player = target:Name()}
        ))
    end
})
```

## Best Practices

### ✅ DO

- Use `ix.util.Include()` and `ix.util.IncludeDir()` for all file loading
- Use `ix.util.SanitizeType()` to validate user input
- Use `ix.util.GetCharacters()` when iterating characters
- Use `ix.util.GetMaterial()` to cache materials
- Use `ix.util.FindPlayer()` for finding players by name
- Use `ix.util.DrawBlur()` for blurred backgrounds in UI
- Use `ix.util.GetStringTime()` for parsing time inputs

### ❌ DON'T

- Don't use `include()` directly for shared/client files
- Don't manually call `AddCSLuaFile()` - use `ix.util.Include()`
- Don't create `Material()` every frame - use `ix.util.GetMaterial()`
- Don't manually iterate players checking for characters
- Don't use `table.Merge()` on tables with metatables
- Don't write custom type validation when `ix.util.SanitizeType()` exists

## Common Patterns

### Pattern 1: Loading Plugin Files

```lua
-- In your plugin's sh_plugin.lua
function PLUGIN:OnLoaded()
    -- Include plugin subdirectories
    ix.util.IncludeDir("plugins/" .. self.uniqueID .. "/commands")
    ix.util.IncludeDir("plugins/" .. self.uniqueID .. "/derma")
    ix.util.IncludeDir("plugins/" .. self.uniqueID .. "/libs")
end
```

### Pattern 2: Validating Command Arguments

```lua
function COMMAND:OnRun(client, arguments)
    -- Find player with fuzzy matching
    local target = ix.util.FindPlayer(arguments[1])

    if not target then
        return "@playerNotFound"
    end

    -- Sanitize numeric input
    local amount = ix.util.SanitizeType(ix.type.number, arguments[2])

    if amount <= 0 then
        return "Invalid amount!"
    end

    -- Use validated inputs
    target:GetCharacter():GiveMoney(amount)
end
```

### Pattern 3: Blurred UI Panel

```lua
local PANEL = {}

function PANEL:Init()
    self:SetSize(500, 400)
    self:Center()
    self:MakePopup()
end

function PANEL:Paint(w, h)
    -- Draw blur effect
    ix.util.DrawBlur(self, 5)

    -- Semi-transparent background
    surface.SetDrawColor(0, 0, 0, 200)
    surface.DrawRect(0, 0, w, h)

    -- Title with shadow
    ix.util.DrawText(
        "My Panel",
        w / 2, 10,
        color_white,
        TEXT_ALIGN_CENTER,
        TEXT_ALIGN_TOP,
        "ixTitleFont"
    )
end

vgui.Register("ixMyPanel", PANEL, "DFrame")
```

## Common Issues

### Issue: Files not loading on client

**Cause**: Using `include()` instead of `ix.util.Include()` for shared/client files
**Fix**: Always use `ix.util.Include()` which automatically calls `AddCSLuaFile()`

```lua
-- WRONG
include("cl_mypanel.lua")

-- CORRECT
ix.util.Include("cl_mypanel.lua")
```

### Issue: FindPlayer returns nil for valid name

**Cause**: Name matching is case-sensitive or exact match expected
**Fix**: `ix.util.FindPlayer()` already does fuzzy matching, ensure name isn't empty

```lua
local target = ix.util.FindPlayer(arguments[1])

if not target then
    return "@playerNotFound"
end
```

### Issue: Material created every frame causing lag

**Cause**: Using `Material()` in `Paint()` hook
**Fix**: Use `ix.util.GetMaterial()` for automatic caching

```lua
-- WRONG
function PANEL:Paint(w, h)
    local mat = Material("icon16/user.png")  -- Created every frame!
end

-- CORRECT
function PANEL:Paint(w, h)
    local mat = ix.util.GetMaterial("icon16/user.png")  -- Cached!
end
```

## See Also

- [Storage Library](storage.md) - For persistent data storage
- [Commands System](../systems/commands.md) - Using ix.util functions in commands
- [Items System](../systems/items.md) - Item validation with ix.util
- [Character System](../systems/character.md) - Using ix.util.GetCharacters()
- Source: `gamemode/core/sh_util.lua`
