# Plugin System

The Helix plugin system is a powerful and flexible way to extend the framework without modifying core files. Plugins can add new features, modify existing behavior, and integrate with all framework systems.

## Overview

### What is a Plugin?

A plugin is a self-contained module that extends Helix functionality. Plugins can:

- Register new items, factions, classes, and attributes
- Add custom commands and chat types
- Create UI components and menus
- Hook into framework events
- Store persistent data
- Add custom entities and weapons
- Provide localization for multiple languages

### Plugin vs Schema

- **Schema**: Your gamemode (e.g., HL2 RP, Fallout RP). Contains your unique setting, content, and rules.
- **Plugin**: Reusable functionality that can work across different schemas (e.g., vendor system, doors, stamina).

```
Schema (Your Gamemode)
├── Core framework functionality
├── Schema-specific content (factions, items, classes)
└── Plugins (add-on features)
    ├── Base plugins (included with Helix)
    └── Custom plugins (your additions)
```

## Plugin Loading Order

```
Server Start
│
├─> Load Helix Framework
│   └─> gamemode/core/libs/sh_plugin.lua
│
├─> ix.plugin.Initialize()
│   │
│   ├─> 1. Load "helix/plugins"
│   │      └─> Base plugins that come with framework
│   │
│   ├─> 2. Load "schema" (if derived gamemode)
│   │      ├─> schema/sh_schema.lua
│   │      └─> Schema acts like a special plugin
│   │
│   ├─> 3. Load "gamemode/plugins"
│   │      └─> Schema-specific plugins
│   │
│   └─> 4. Fire "InitializedPlugins" hook
│          └─> All plugins are now loaded
│
└─> Server Ready
```

## Plugin Structure

### Basic Plugin

**Minimum required structure:**

```
plugins/myplugin/
└── sh_plugin.lua          # Main plugin file
```

**sh_plugin.lua:**
```lua
PLUGIN.name = "My Plugin"
PLUGIN.author = "Your Name"
PLUGIN.description = "What your plugin does"

-- Add your code here
function PLUGIN:PlayerSpawn(player)
    player:ChatPrint("Welcome!")
end
```

### Complete Plugin Structure

**Full-featured plugin:**

```
plugins/myplugin/
├── sh_plugin.lua          # Main plugin file (required)
├── sh_commands.lua        # Command definitions
├── sh_config.lua          # Configuration options
├── sv_plugin.lua          # Server-only code
├── sv_hooks.lua           # Server hooks
├── cl_plugin.lua          # Client-only code
├── cl_hooks.lua           # Client hooks
├── items/                 # Item definitions
│   ├── sh_item1.lua
│   └── sh_item2.lua
├── factions/              # Faction definitions
│   └── sh_faction1.lua
├── classes/               # Class definitions
│   └── sh_class1.lua
├── attributes/            # Attribute definitions
│   └── sh_attribute1.lua
├── derma/                 # UI components
│   ├── cl_menu.lua
│   └── cl_panel.lua
├── entities/              # Custom entities
│   ├── entities/
│   │   └── ent_custom.lua
│   └── weapons/
│       └── weapon_custom.lua
├── languages/             # Localizations
│   ├── sh_english.lua
│   └── sh_spanish.lua
└── libs/                  # Custom libraries
    └── sh_mylib.lua
```

## Plugin Metadata

### Required Fields

```lua
PLUGIN.name = "My Plugin"           -- Display name
PLUGIN.author = "Your Name"          -- Your name/organization
PLUGIN.description = "Description"   -- What the plugin does
```

### Optional Fields

```lua
PLUGIN.version = "1.0.0"            -- Version (semantic versioning)
PLUGIN.license = "MIT"              -- License type
PLUGIN.website = "https://..."      -- Plugin website
PLUGIN.schema = "hl2rp"             -- Specific schema requirement
PLUGIN.depends = {"vendor"}         -- Required plugin dependencies
```

### Accessing Plugin Data

```lua
-- Inside plugin methods, use self
function PLUGIN:MyMethod()
    print(self.name)        -- "My Plugin"
    print(self.author)      -- "Your Name"
    print(self.uniqueID)    -- "myplugin" (folder name)
end

-- From outside, use ix.plugin.list
local myPlugin = ix.plugin.list["myplugin"]
print(myPlugin.name)
```

## Plugin Methods

### Hook Methods

Any function you define on PLUGIN becomes a hook:

```lua
-- This automatically hooks into PlayerSpawn
function PLUGIN:PlayerSpawn(player)
    player:SetHealth(100)
end

-- This hooks into PlayerSay
function PLUGIN:PlayerSay(player, text, team)
    if text == "!hello" then
        return "Hello to you too!"
    end
end

-- This hooks into PlayerDeath
function PLUGIN:PlayerDeath(victim, inflictor, attacker)
    victim:ChatPrint("You died!")
end
```

### Initialization Methods

```lua
function PLUGIN:InitializedPlugins()
    -- Called after ALL plugins are loaded
    -- Good for setup that depends on other plugins
    print("All plugins loaded!")
end

function PLUGIN:InitializedSchema()
    -- Called after schema is loaded
    -- Good for schema-specific initialization
end

function PLUGIN:OnPluginLoaded(plugin)
    -- Called when ANY plugin loads
    -- Receive the loaded plugin as parameter
    print("Plugin loaded:", plugin.name)
end
```

### Cleanup Methods

```lua
function PLUGIN:OnPluginUnloaded()
    -- Called when this plugin is unloaded
    -- Clean up timers, entities, UI, etc.

    timer.Remove("MyPluginTimer")

    if CLIENT and IsValid(self.menu) then
        self.menu:Remove()
    end
end
```

## File Loading

### Manual File Inclusion

```lua
-- In sh_plugin.lua
ix.util.Include("sh_config.lua")       -- Loads config file
ix.util.Include("sv_hooks.lua")        -- Server hooks
ix.util.Include("cl_hooks.lua")        -- Client hooks
ix.util.IncludeDir("libs")             -- Load all files in libs/
```

### Automatic Directory Loading

These directories are loaded automatically:

- `languages/` - Localization files
- `items/` - Item definitions
- `factions/` - Faction definitions
- `classes/` - Class definitions
- `attributes/` - Attribute definitions
- `derma/` - UI components (client-only)
- `entities/` - Custom entities
- `plugins/` - Nested plugins

### Loading Order

```lua
-- 1. Languages loaded first
languages/sh_english.lua

-- 2. Libraries loaded
libs/sh_mylib.lua

-- 3. Game content loaded
attributes/sh_strength.lua
factions/sh_rebels.lua
classes/sh_medic.lua
items/sh_medkit.lua

-- 4. Main plugin file
sh_plugin.lua

-- 5. Realm-specific files
sv_plugin.lua (server)
cl_plugin.lua (client)

-- 6. Derma loaded (client)
derma/cl_menu.lua

-- 7. Entities loaded last
entities/entities/ent_custom.lua
```

## Data Persistence

### Saving Plugin Data

```lua
function PLUGIN:SaveMyData()
    self:SetData({
        savedValue = self.myValue,
        savedTable = self.myTable,
        timestamp = os.time()
    })
end
```

### Loading Plugin Data

```lua
function PLUGIN:InitializedPlugins()
    local data = self:GetData() or {}

    self.myValue = data.savedValue or 100
    self.myTable = data.savedTable or {}
    self.timestamp = data.timestamp or 0
end
```

### Automatic Saving

Data is automatically saved when the plugin is unloaded or server shuts down.

### Manual Save

```lua
function PLUGIN:PlayerDisconnected(client)
    -- Force save when needed
    self:SaveMyData()
end
```

## Plugin Variables

### Instance Variables

```lua
function PLUGIN:InitializedPlugins()
    -- Store data on the plugin instance
    self.playerData = {}
    self.timer = 0
    self.enabled = true
end

function PLUGIN:UsePlayerData(client)
    -- Access instance variables with self
    self.playerData[client:SteamID()] = {
        score = 100
    }
end
```

### Shared Variables

```lua
-- In sh_plugin.lua (runs on both client and server)
PLUGIN.sharedValue = 100

function PLUGIN:GetSharedValue()
    return self.sharedValue
end
```

### Realm-Specific Variables

```lua
-- Server-only
if SERVER then
    PLUGIN.serverData = {}
end

-- Client-only
if CLIENT then
    PLUGIN.clientData = {}
end
```

## Hook Caching

The plugin system automatically caches hooks for performance:

```lua
-- When plugin loads:
PLUGIN.MyHook = function(self, arg1, arg2)
    -- Your code
end

-- Helix automatically does:
ix.plugin.HOOKS_CACHE["MyHook"] = ix.plugin.HOOKS_CACHE["MyHook"] or {}
table.insert(ix.plugin.HOOKS_CACHE["MyHook"], {
    plugin = PLUGIN,
    method = PLUGIN.MyHook
})

-- When hook is called:
function ix.plugin.Call(hookName, ...)
    local cache = ix.plugin.HOOKS_CACHE[hookName]
    if cache then
        for _, data in ipairs(cache) do
            local result = data.method(data.plugin, ...)
            if result ~= nil then
                return result  -- First non-nil return value stops chain
            end
        end
    end
    return hook.Run(hookName, ...)  -- Fall through to GMod hooks
end
```

## Available Hooks

### Character Hooks

```lua
function PLUGIN:CharacterLoaded(character)
    -- Called when character data is loaded from database
end

function PLUGIN:PlayerLoadedCharacter(client, character, lastChar)
    -- Called when player selects a character
end

function PLUGIN:CanPlayerCreateCharacter(client)
    -- Return false to prevent character creation
end

function PLUGIN:OnCharacterCreated(client, character)
    -- Called after character is created
end

function PLUGIN:CharacterDeleted(client, character)
    -- Called when character is deleted
end
```

### Item Hooks

```lua
function PLUGIN:CanPlayerInteractItem(client, action, item)
    -- Return false to prevent item interaction
end

function PLUGIN:OnItemTransferred(item, oldInventory, newInventory)
    -- Called when item moves between inventories
end

function PLUGIN:OnCharacterInventoryUpdated(character, inventory)
    -- Called when character's inventory changes
end
```

### Combat Hooks

```lua
function PLUGIN:PlayerDeath(victim, inflictor, attacker)
    -- Called when player dies
end

function PLUGIN:GetPlayerPainSound(client)
    -- Return sound path to override pain sound
end

function PLUGIN:PlayerHurt(client, attacker, health, damage)
    -- Called when player takes damage
end

function PLUGIN:CanPlayerTakeDamage(client, attacker)
    -- Return false to prevent damage
end
```

### Networking Hooks

```lua
function PLUGIN:PlayerInitialSpawn(client)
    -- Called when player first joins (before character selection)
end

function PLUGIN:PlayerDisconnected(client)
    -- Called when player disconnects
end

function PLUGIN:PlayerSay(client, text, team)
    -- Called when player sends chat message
    -- Return string to modify text
    -- Return "" to block message
end
```

### UI Hooks (Client)

```lua
function PLUGIN:HUDPaint()
    -- Called every frame to draw HUD
end

function PLUGIN:CreateMenuButtons(tabs)
    -- Add buttons to F1 menu
    tabs["mybutton"] = function()
        -- Create your menu
    end
end

function PLUGIN:CharacterMenuCreated(panel)
    -- Called when character menu is created
end
```

### For complete hook list, see [Hook Reference](hooks.md)

## Plugin Communication

### Calling Other Plugin Methods

```lua
-- Get plugin reference
local vendorPlugin = ix.plugin.list["vendor"]

-- Check if plugin exists
if vendorPlugin then
    -- Call plugin method
    vendorPlugin:DoSomething()
end
```

### Plugin Dependencies

```lua
PLUGIN.depends = {"vendor", "doors"}  -- This plugin requires vendor and doors

function PLUGIN:InitializedPlugins()
    -- Dependencies are guaranteed to be loaded
    local vendor = ix.plugin.list["vendor"]
    -- vendor is guaranteed to exist
end
```

### Broadcasting to All Plugins

```lua
-- Using hooks
hook.Run("MyCustomHook", arg1, arg2)

-- All plugins can listen
function PLUGIN:MyCustomHook(arg1, arg2)
    -- Respond to custom hook
end
```

## Conditional Loading

### Prevent Plugin from Loading

```lua
-- In sh_plugin.lua, before PLUGIN is defined
if not file.Exists("data/mymodule.txt", "GAME") then
    return  -- Don't load plugin
end

PLUGIN.name = "My Plugin"
-- Rest of plugin code
```

### Schema-Specific Plugin

```lua
PLUGIN.schema = "hl2rp"  -- Only load on hl2rp schema

-- Or check manually:
if Schema.uniqueID != "hl2rp" then
    return
end
```

### Using PluginShouldLoad Hook

```lua
-- In another plugin or schema
function PLUGIN:PluginShouldLoad(uniqueID)
    if uniqueID == "pluginname" and not self.allowPlugin then
        return false  -- Prevent loading
    end
end
```

## Creating Commands in Plugins

```lua
ix.command.Add("MyCommand", {
    description = "Does something cool",
    arguments = {
        ix.type.player,  -- First argument: player
        ix.type.number   -- Second argument: number
    },
    OnRun = function(self, client, target, amount)
        target:ChatPrint("You received " .. amount .. " points!")
        return "Gave " .. amount .. " points to " .. target:Name()
    end
})
```

See [Command System](../systems/commands.md) for more details.

## Creating Configuration Options

```lua
ix.config.Add("myPluginEnabled", true, "Enable my plugin features", nil, {
    category = "My Plugin"
})

ix.config.Add("myPluginValue", 100, "Value for calculations", function(oldValue, newValue)
    -- Called when value changes
    print("Value changed from", oldValue, "to", newValue)
end, {
    data = {min = 0, max = 1000},
    category = "My Plugin"
})

-- Access config
if ix.config.Get("myPluginEnabled") then
    local value = ix.config.Get("myPluginValue")
end
```

See [Configuration System](../systems/configuration.md) for more details.

## Example Plugin Templates

### Simple Plugin

```lua
-- plugins/simpleexample/sh_plugin.lua
PLUGIN.name = "Simple Example"
PLUGIN.author = "Your Name"
PLUGIN.description = "A simple example plugin"

function PLUGIN:PlayerSay(client, text)
    if text == "!time" then
        return "Server time: " .. os.date("%H:%M:%S")
    end
end
```

### Medium Complexity Plugin

```lua
-- plugins/pointsystem/sh_plugin.lua
PLUGIN.name = "Point System"
PLUGIN.author = "Your Name"
PLUGIN.description = "Adds a point tracking system"

if SERVER then
    function PLUGIN:InitializedPlugins()
        self:LoadData()
        self.points = self:GetData().points or {}
    end

    function PLUGIN:SavePoints()
        self:SetData({points = self.points})
    end

    function PLUGIN:AddPoints(client, amount)
        local steamID = client:SteamID()
        self.points[steamID] = (self.points[steamID] or 0) + amount
        self:SavePoints()
    end

    function PLUGIN:GetPoints(client)
        return self.points[client:SteamID()] or 0
    end
end

-- Command
ix.command.Add("GivePoints", {
    description = "Give points to a player",
    adminOnly = true,
    arguments = {
        ix.type.player,
        ix.type.number
    },
    OnRun = function(self, client, target, amount)
        PLUGIN:AddPoints(target, amount)
        return "Gave " .. amount .. " points to " .. target:Name()
    end
})
```

### Complex Plugin with UI

See [Creating Plugins Guide](creating-plugins.md) for a complete walkthrough.

## Best Practices

### Do's

- ✅ Use PLUGIN table for all methods
- ✅ Check realm before using realm-specific functions
- ✅ Save plugin data with SetData/GetData
- ✅ Clean up resources in OnPluginUnloaded
- ✅ Use configuration options for customizable values
- ✅ Document your hooks and methods
- ✅ Test with multiple players
- ✅ Handle nil cases properly

### Don'ts

- ❌ Don't pollute global namespace
- ❌ Don't modify core framework files
- ❌ Don't trust client input
- ❌ Don't use global variables for plugin state
- ❌ Don't forget to check if entities are valid
- ❌ Don't create memory leaks (unclosed panels, timers)
- ❌ Don't use expensive operations in hooks like Think

## Debugging Plugins

### Print Debugging

```lua
function PLUGIN:MyMethod(arg)
    print("MyMethod called with:", arg)
    PrintTable(arg)  -- Print table contents
end
```

### Conditional Debugging

```lua
PLUGIN.debug = true

function PLUGIN:DebugPrint(...)
    if self.debug then
        print("[" .. self.name .. "]", ...)
    end
end

function PLUGIN:MyMethod()
    self:DebugPrint("Method called")
end
```

### Error Catching

```lua
function PLUGIN:RiskyOperation()
    local success, err = pcall(function()
        -- Risky code here
    end)

    if not success then
        ErrorNoHalt("[" .. self.name .. "] Error: " .. tostring(err) .. "\n")
    end
end
```

## See Also

- [Creating Plugins](creating-plugins.md) - Step-by-step guide
- [Plugin Structure](plugin-structure.md) - Detailed structure
- [Hook System](hooks.md) - All available hooks
- [Plugin Best Practices](best-practices.md) - Guidelines
- [Example Plugins](../examples/) - Working examples
