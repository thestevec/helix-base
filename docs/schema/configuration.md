# Schema Configuration

> **Reference**: `schema/sh_schema.lua`

Schema configuration defines your gamemode's metadata, settings, and initialization logic.

## ⚠️ Important: Use Schema Configuration System

**Always use the schema configuration table** rather than creating custom initialization systems. The framework provides:
- Automatic schema metadata loading
- Configuration variable management
- Schema-specific hooks
- Integration with core systems

## Core Concepts

### What is Schema Configuration?

Schema configuration is the central setup file that:
- Defines schema metadata (name, author, description)
- Sets up schema-specific settings
- Initializes schema systems
- Includes additional schema files

### Schema vs Plugin Configuration

- **Schema**: Core gamemode settings that define your RP setting
- **Plugin**: Optional features that can be toggled on/off

## Required Schema Files

### gamemode/init.lua

**Reference**: Derive from Helix gamemode

```lua
-- gamemode/init.lua (SERVER)
AddCSLuaFile("cl_init.lua")
DeriveGamemode("helix")
```

**⚠️ Do NOT**: Add any other code to this file. It only derives from Helix.

### gamemode/cl_init.lua

**Reference**: Client entry point

```lua
-- gamemode/cl_init.lua (CLIENT)
DeriveGamemode("helix")
```

**⚠️ Do NOT**: Add any other code here either.

### schema/sh_schema.lua

**Reference**: Main schema configuration (REQUIRED)

```lua
-- schema/sh_schema.lua
Schema.name = "My Roleplay"
Schema.author = "Your Name"
Schema.description = "A custom roleplay schema"

-- Optional: Include other schema files
ix.util.Include("sv_schema.lua")
ix.util.Include("cl_schema.lua")
```

## Schema Properties

### Required Properties

```lua
-- schema/sh_schema.lua
Schema.name = "Schema Name"          -- Display name (REQUIRED)
Schema.author = "Author Name"        -- Creator name (REQUIRED)
Schema.description = "Description"   -- Brief description (REQUIRED)
```

**⚠️ If missing**: Schema will still load but show default values.

### Optional Properties

```lua
Schema.version = "1.0.0"             -- Version number
Schema.discord = "discord.gg/link"   -- Discord invite link
Schema.website = "https://example.com" -- Website URL
```

### Automatic Properties

**Reference**: `gamemode/core/libs/sh_plugin.lua:470`

Framework automatically adds:
```lua
Schema.folder = "myschema"           -- Schema folder name
Schema.uniqueID = "myschema"         -- Same as folder
Schema.isHL2RP = false               -- Set to true for HL2RP schemas
```

## Configuration Variables

### Using ix.config

**Reference**: `gamemode/core/libs/sh_config.lua`

Schema can define custom configuration variables:

```lua
-- schema/sh_schema.lua
ix.config.Add("mySchemaOption", true, "Enable custom feature", function(oldValue, newValue)
    print("mySchemaOption changed from", oldValue, "to", newValue)
end, {
    category = "schema"
})

ix.config.Add("maxSalary", 500, "Maximum salary amount", nil, {
    data = {min = 0, max = 10000},
    category = "schema"
})

ix.config.Add("startingMoney", 100, "Starting money for new characters", nil, {
    data = {min = 0, max = 1000},
    category = "schema"
})
```

### Getting Config Values

```lua
-- Anywhere in schema/plugins
local value = ix.config.Get("mySchemaOption", false)

if value then
    -- Feature enabled
end

local startMoney = ix.config.Get("startingMoney", 100)
```

## Schema Initialization

### Including Schema Files

**Reference**: Schema file inclusion

```lua
-- schema/sh_schema.lua
Schema.name = "My Schema"
Schema.author = "Me"
Schema.description = "My awesome schema"

-- Include server file
if SERVER then
    ix.util.Include("sv_schema.lua")
end

-- Include client file
if CLIENT then
    ix.util.Include("cl_schema.lua")
end

-- Or include both (auto-sends to client)
ix.util.Include("sv_schema.lua")
ix.util.Include("cl_schema.lua")
```

### Server Schema File

```lua
-- schema/sv_schema.lua
-- Server-only initialization

function Schema:InitPostEntity()
    -- Called after entities are initialized
    print("Server initialized!")
end

function Schema:PlayerLoadedCharacter(client, character, lastChar)
    -- Give starting money
    if not character:GetData("receivedStartingMoney") then
        local startMoney = ix.config.Get("startingMoney", 100)
        character:GiveMoney(startMoney)
        character:SetData("receivedStartingMoney", true)
    end
end
```

### Client Schema File

```lua
-- schema/cl_schema.lua
-- Client-only initialization

function Schema:HUDPaint()
    -- Custom HUD drawing
end

function Schema:OnCharacterMenuCreated(panel)
    -- Customize character menu
end
```

## Auto-Loading Directories

**Reference**: `schema/` folder structure

Framework automatically loads files from:

### schema/items/

Place item files with `sh_` prefix:

```
schema/items/
├── sh_medkit.lua
├── sh_weapon_pistol.lua
└── sh_food_bread.lua
```

### schema/factions/

Place faction files with `sh_` prefix:

```
schema/factions/
├── sh_citizen.lua
├── sh_police.lua
└── sh_resistance.lua
```

### schema/classes/

Place class files with `sh_` prefix:

```
schema/classes/
├── sh_officer.lua
├── sh_chief.lua
└── sh_doctor.lua
```

### schema/commands/

Optional custom commands:

```
schema/commands/
└── sh_commands.lua
```

### schema/derma/

Optional custom UI:

```
schema/derma/
├── cl_menu.lua
└── cl_custom_panel.lua
```

### schema/languages/

Optional localization:

```
schema/languages/
├── sh_english.lua
└── sh_spanish.lua
```

## Complete Example

### Minimal Schema

```lua
-- schema/sh_schema.lua
Schema.name = "Half-Life 2 Roleplay"
Schema.author = "Community"
Schema.description = "A roleplay schema set in the Half-Life 2 universe"
Schema.version = "1.0.0"
```

### Full Schema Configuration

```lua
-- schema/sh_schema.lua
Schema.name = "Dark RP"
Schema.author = "Your Name"
Schema.description = "A custom dark roleplay schema"
Schema.version = "2.1.0"
Schema.discord = "discord.gg/example"
Schema.website = "https://example.com"

-- Include additional files
ix.util.Include("sv_schema.lua")
ix.util.Include("cl_schema.lua")

-- Configuration options
ix.config.Add("salaryEnabled", true, "Enable automatic salary", function(oldValue, newValue)
    if newValue then
        timer.Create("SchemaSalary", 300, 0, function()
            Schema:PaySalaries()
        end)
    else
        timer.Remove("SchemaSalary")
    end
end, {
    category = "schema"
})

ix.config.Add("salaryAmount", 50, "Base salary amount", nil, {
    data = {min = 0, max = 1000},
    category = "schema"
})

ix.config.Add("startingMoney", 100, "Starting money", nil, {
    data = {min = 0, max = 10000},
    category = "schema"
})

ix.config.Add("enableVoiceChat", true, "Enable proximity voice", nil, {
    category = "schema"
})

-- Custom functions
function Schema:PaySalaries()
    for _, client in ipairs(player.GetAll()) do
        local character = client:GetCharacter()
        if character then
            local faction = ix.faction.indices[character:GetFaction()]
            local amount = faction.pay or ix.config.Get("salaryAmount", 50)

            if amount > 0 then
                character:GiveMoney(amount)
                client:Notify("Received salary: " .. ix.currency.Get(amount))
            end
        end
    end
end
```

### schema/sv_schema.lua

```lua
-- Server initialization

function Schema:InitPostEntity()
    -- Start salary timer if enabled
    if ix.config.Get("salaryEnabled") then
        timer.Create("SchemaSalary", 300, 0, function()
            self:PaySalaries()
        end)
    end
end

function Schema:PlayerLoadedCharacter(client, character, lastChar)
    -- First time setup
    if not character:GetData("initialized") then
        local startMoney = ix.config.Get("startingMoney", 100)
        character:GiveMoney(startMoney)
        character:SetData("initialized", true)

        client:ChatPrint("Welcome to " .. self.name .. "!")
    end
end

function Schema:PlayerSpawn(client)
    -- Set default speeds
    client:SetRunSpeed(240)
    client:SetWalkSpeed(100)
    client:SetJumpPower(200)
end
```

### schema/cl_schema.lua

```lua
-- Client initialization

function Schema:InitPostEntity()
    print("Client loaded schema:", self.name)
end

function Schema:HUDPaint()
    -- Custom HUD elements
    local client = LocalPlayer()
    local char = client:GetCharacter()

    if char then
        -- Draw custom info
    end
end

function Schema:OnCharacterMenuCreated(panel)
    -- Customize character menu
    panel:SetTitle(self.name .. " - Character Selection")
end
```

## Best Practices

### ✅ DO

- Set all required properties (name, author, description)
- Use `ix.config.Add()` for configurable settings
- Keep `sh_schema.lua` clean and readable
- Organize code into separate files (sv_schema.lua, cl_schema.lua)
- Use schema hooks for game logic
- Document your config options
- Use categories for organization

### ❌ DON'T

- Don't add code to gamemode/init.lua or gamemode/cl_init.lua
- Don't modify Helix core files for schema settings
- Don't hardcode values that should be configurable
- Don't put everything in one file
- Don't create custom loading systems
- Don't bypass ix.config for settings

## Common Patterns

### Faction-Based Starting Items

```lua
function Schema:PlayerLoadedCharacter(client, character, lastChar)
    if not character:GetData("receivedStartingItems") then
        local faction = ix.faction.indices[character:GetFaction()]
        local inventory = character:GetInventory()

        -- Give faction-specific items
        if faction.uniqueID == "citizen" then
            inventory:Add("item_phone")
        elseif faction.uniqueID == "police" then
            inventory:Add("item_radio")
            inventory:Add("item_handcuffs")
        end

        character:SetData("receivedStartingItems", true)
    end
end
```

### Time-Based Events

```lua
function Schema:InitPostEntity()
    -- Daily reset at midnight
    timer.Create("SchemaDailyReset", 60, 0, function()
        local hour = tonumber(os.date("%H"))

        if hour == 0 then
            self:DailyReset()
        end
    end)
end

function Schema:DailyReset()
    -- Reset daily quests, salaries, etc.
    for _, client in ipairs(player.GetAll()) do
        local char = client:GetCharacter()
        if char then
            char:SetData("dailyQuestDone", false)
        end
    end
end
```

### Custom Config Categories

```lua
-- Organize configs by category
ix.config.Add("economyInflation", 1.0, "Economy inflation rate", nil, {
    data = {min = 0.1, max = 10.0, decimals = 2},
    category = "economy"
})

ix.config.Add("economyStartMoney", 500, "Starting money", nil, {
    data = {min = 0, max = 10000},
    category = "economy"
})

ix.config.Add("combatEnabled", true, "Enable PvP combat", nil, {
    category = "gameplay"
})

ix.config.Add("combatDamageMultiplier", 1.0, "Damage multiplier", nil, {
    data = {min = 0.1, max = 5.0, decimals = 1},
    category = "gameplay"
})
```

## Common Issues

### Schema not loading

**Cause**: Missing required files or wrong folder structure
**Fix**: Ensure you have:
- gamemode/init.lua
- gamemode/cl_init.lua
- schema/sh_schema.lua
- Correct `DeriveGamemode("helix")` calls

### Config changes not saving

**Cause**: Config not properly defined
**Fix**: Use `ix.config.Add()` with correct parameters and category

### Files not auto-loading

**Cause**: Wrong file naming or location
**Fix**: Place files in correct directories with `sh_` prefix

## See Also

- [Schema Structure](structure.md) - Directory organization
- [Configuration System](../systems/configuration.md) - Config system details
- [Plugin System](../plugins/plugin-system.md) - Plugin configuration
- Source: `gamemode/core/libs/sh_plugin.lua`
- Source: `gamemode/core/libs/sh_config.lua`
