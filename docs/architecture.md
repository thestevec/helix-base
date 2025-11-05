# Helix Framework Architecture

## Overview

Helix is a modular roleplay framework built on top of Garry's Mod's Sandbox gamemode. It provides a comprehensive foundation for creating roleplay gamemodes with systems for character management, inventory, items, factions, and extensive customization capabilities.

### Key Statistics

- **Total Lua files**: 163
- **Core framework lines**: 33,411
- **Core libraries**: 29
- **Derma UI components**: 30
- **Meta table extensions**: 6
- **Base plugins**: 18+
- **Supported languages**: 10

### Design Philosophy

1. **Modularity**: Systems are separated into discrete libraries with clear responsibilities
2. **Extensibility**: Plugin system allows adding/modifying functionality without touching core
3. **Data-driven**: Items, factions, classes use data structures rather than hardcoded logic
4. **Client-server separation**: Clear distinction between client, server, and shared code
5. **Database abstraction**: Supports both SQLite and MySQL transparently
6. **Performance**: Hook caching, efficient network usage, optimized UI rendering

## Directory Structure

```
helix-base/
├── gamemode/
│   ├── init.lua                    # Server entry point
│   ├── cl_init.lua                 # Client entry point
│   ├── shared.lua                  # Shared initialization (158 lines)
│   ├── core/
│   │   ├── libs/                   # Core library system (29 files)
│   │   │   ├── sh_*.lua           # Shared libraries
│   │   │   ├── cl_*.lua           # Client-only libraries
│   │   │   ├── sv_*.lua           # Server-only libraries
│   │   │   └── thirdparty/        # Third-party libraries
│   │   ├── meta/                   # Meta table extensions (6 files)
│   │   ├── derma/                  # UI components (30 files)
│   │   └── hooks/                  # Hook implementations
│   ├── config/
│   │   ├── sh_config.lua          # Framework configuration
│   │   └── sh_options.lua         # User options
│   ├── items/
│   │   ├── base/                  # Base item classes
│   │   ├── ammo/                  # Ammunition items
│   │   ├── weapons/               # Weapon items
│   │   ├── bags/                  # Container items
│   │   └── pacoutfit/             # PAC3 outfits
│   └── languages/                  # Localization files (10 languages)
├── plugins/                        # Base plugins
├── schema/                         # Active schema (if derived gamemode)
├── entities/                       # Custom entities
│   ├── entities/
│   │   ├── ix_item.lua            # Dropped item entity
│   │   ├── ix_money.lua           # Money entity
│   │   └── ix_shipment.lua        # Shipment entity
│   └── weapons/
│       └── ix_hands.lua           # Unarmed weapon
├── docs/                           # Documentation
└── helix.yml                       # Database configuration
```

## Initialization Sequence

### Server-Side Boot Process

```lua
1. gamemode/init.lua loaded
   ├── DeriveGamemode("sandbox")
   ├── Create global ix table
   ├── AddCSLuaFile() for client files
   └── Include core/sh_util.lua

2. gamemode/shared.lua included
   ├── Set gamemode metadata (name, author, version)
   ├── Override player model functions
   ├── Include core libraries in order:
   │   ├── Utility functions (sh_util)
   │   ├── Data system (sh_data)
   │   ├── All other libraries (sh_*.lua)
   │   ├── Server libraries (sv_*.lua)
   │   ├── Meta tables
   │   └── Hooks
   ├── Load languages
   └── Load base items

3. GM:Initialize() hook fired
   ├── ix.plugin.Initialize()
   │   ├── Load "helix/plugins" directory
   │   ├── Load "schema" (if derived gamemode)
   │   │   └── Load schema/sh_schema.lua
   │   ├── Load "gamemode/plugins" directory
   │   └── Fire "InitializedPlugins" hook
   ├── ix.option.Load() - Restore saved options
   ├── ix.config.Load() - Restore configurations
   └── Initialize database connection

4. Database initialization
   ├── Load helix.yml or use SQLite
   ├── Create/verify tables:
   │   ├── ix_players (player data)
   │   ├── ix_characters (character data)
   │   ├── ix_inventories (inventory data)
   │   └── Plugin-specific tables
   └── Fire "DatabaseConnected" hook

5. Map-specific initialization
   ├── Load saved spawn points
   ├── Load saved doors
   ├── Load saved storage containers
   └── Fire "InitPostEntity" hook
```

### Client-Side Boot Process

```lua
1. gamemode/cl_init.lua loaded
   ├── Font overrides for non-Windows
   ├── DeriveGamemode("sandbox")
   └── Create global ix table

2. gamemode/shared.lua included
   ├── Same as server (shared portion)
   └── Include client libraries (cl_*.lua)

3. Receive files from server
   └── All AddCSLuaFile() files downloaded

4. Core framework initialization
   ├── Load libraries
   ├── Load meta tables
   ├── Load Derma components
   ├── Load hooks
   └── Load languages and items

5. Plugin initialization (client-side)
   ├── Load plugin UI components
   ├── Register client hooks
   └── Initialize client-side state

6. UI initialization
   ├── Create main menu
   ├── Initialize HUD
   └── Setup notification system
```

### Plugin Loading Sequence

```lua
ix.plugin.Load(uniqueID, path, isSingleFile, variable)
  │
  ├── 1. Check "PluginShouldLoad" hook
  │      └── Return false to prevent loading
  │
  ├── 2. Create plugin table
  │      PLUGIN = {
  │          uniqueID = "pluginname",
  │          name = "Plugin Name",
  │          author = "Author",
  │          description = "Description"
  │      }
  │
  ├── 3. Load subdirectories (if not single file):
  │      ├── languages/     # Load localization
  │      ├── libs/          # Load custom libraries
  │      ├── attributes/    # Register attributes
  │      ├── factions/      # Register factions
  │      ├── classes/       # Register classes
  │      ├── items/         # Register items
  │      ├── plugins/       # Load nested plugins
  │      ├── derma/         # Load UI components
  │      └── entities/      # Load custom entities
  │
  ├── 4. Load main plugin file
  │      ├── sh_plugin.lua (shared)
  │      ├── cl_plugin.lua (client, if exists)
  │      └── sv_plugin.lua (server, if exists)
  │
  ├── 5. Cache plugin hooks
  │      └── Scan plugin methods and register as hooks
  │
  ├── 6. Fire "PluginLoaded" hook
  │      └── OnPluginLoaded(plugin)
  │
  └── 7. Store in ix.plugin.list[uniqueID]
```

## Core Systems Architecture

### Global Namespace Structure

```lua
ix = {
    -- Core libraries
    util = {},         # Utility functions
    data = {},         # Data storage
    storage = {},      # Persistent storage
    db = {},           # Database abstraction
    net = {},          # Networking

    -- Game systems
    char = {},         # Character system
    item = {},         # Item system
    inventory = {},    # Inventory management
    faction = {},      # Faction system
    class = {},        # Class system
    attribs = {},      # Attributes

    -- Gameplay features
    command = {},      # Command system
    chatbox = {},      # Chat system
    config = {},       # Configuration
    option = {},       # User options
    currency = {},     # Money/currency
    business = {},     # Business ownership
    flag = {},         # Permission flags

    -- UI and display
    bar = {},          # HUD bars
    hud = {},          # HUD management
    markup = {},       # Text formatting
    menu = {},         # Menu system
    notice = {},       # Notifications

    -- Framework management
    plugin = {},       # Plugin system
    log = {},          # Logging
    language = {},     # Localization
    date = {},         # Date/time utilities
    anim = {},         # Animations

    -- Meta tables
    meta = {
        player = {},
        character = {},
        item = {},
        inventory = {},
        entity = {}
    }
}
```

### Character Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                    Player Joins Server                       │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│            Query Database for Player's Characters            │
│  SELECT * FROM ix_characters WHERE _steamID = ?              │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Display Character Selection Screen              │
│  - Show existing characters (name, desc, model, faction)     │
│  - Option to create new character (if not at max)            │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
                  ┌─────────┴─────────┐
                  │                   │
                  ▼                   ▼
        ┌─────────────────┐  ┌────────────────┐
        │  Create New      │  │  Select        │
        │  Character       │  │  Existing      │
        └────────┬─────────┘  └───────┬────────┘
                 │                    │
                 ▼                    │
        ┌─────────────────┐           │
        │ Character        │           │
        │ Creation UI      │           │
        │ - Name           │           │
        │ - Description    │           │
        │ - Model          │           │
        │ - Faction        │           │
        │ - Attributes     │           │
        └────────┬─────────┘           │
                 │                    │
                 ▼                    │
        ┌─────────────────┐           │
        │ Insert into DB   │           │
        │ ix_characters    │           │
        └────────┬─────────┘           │
                 │                    │
                 └──────────┬─────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Load Character Data                         │
│  - Create character object (ix.char.New)                     │
│  - Load from database (ix.char.Restore)                      │
│  - Create character instance with all data                   │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Load Character Inventory                    │
│  SELECT * FROM ix_inventories WHERE _characterID = ?         │
│  - Restore inventory grid                                    │
│  - Restore all items with positions                          │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Apply Character to Player                   │
│  player:SetCharacter(character)                              │
│  - Set player model                                          │
│  - Set faction/class                                         │
│  - Set attributes                                            │
│  - Restore equipment                                         │
│  - Set spawn location                                        │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    Fire Hooks                                │
│  - PlayerLoadout(player)                                     │
│  - PostPlayerLoadout(player)                                 │
│  - PlayerLoadedCharacter(player, character)                  │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Player Spawned in Game                      │
└─────────────────────────────────────────────────────────────┘
```

### Inventory System Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    Inventory System                           │
└───────────────────────────┬──────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            │               │               │
            ▼               ▼               ▼
    ┌────────────┐  ┌────────────┐  ┌────────────┐
    │ Character  │  │  Storage   │  │    Item    │
    │ Inventory  │  │ Inventory  │  │  Internal  │
    │            │  │(Container) │  │ Inventory  │
    │ Default:   │  │            │  │  (Bags)    │
    │ 6x4 grid   │  │ Variable   │  │            │
    └─────┬──────┘  └──────┬─────┘  └──────┬─────┘
          │                │                │
          └────────────────┼────────────────┘
                           │
                           ▼
              ┌────────────────────────┐
              │   Inventory Instance   │
              │   ────────────────────  │
              │   Properties:           │
              │   - id                  │
              │   - w, h (grid size)    │
              │   - owner (entity)      │
              │   - typeID              │
              │   ────────────────────  │
              │   Methods:              │
              │   - Add(item, x, y)     │
              │   - Remove(itemID)      │
              │   - CanItemFit(item)    │
              │   - FindFreePosition()  │
              │   - Sync(receiver)      │
              └────────────┬───────────┘
                           │
                           ▼
              ┌────────────────────────┐
              │    Item Instance        │
              │   ────────────────────  │
              │   Properties:           │
              │   - id (unique ID)      │
              │   - uniqueID (type)     │
              │   - data (custom data)  │
              │   - gridX, gridY        │
              │   - gridW, gridH        │
              │   - invID               │
              │   ────────────────────  │
              │   Methods:              │
              │   - Transfer(inv)       │
              │   - Remove()            │
              │   - GetOwner()          │
              │   - SetData(k, v)       │
              └────────────────────────┘
```

### Database Schema

```sql
-- Players table
CREATE TABLE ix_players (
    _steamID VARCHAR(20) PRIMARY KEY,
    _steamName VARCHAR(100),
    _playTime INT DEFAULT 0,
    _data TEXT,  -- JSON encoded data
    _lastJoin INT
);

-- Characters table
CREATE TABLE ix_characters (
    _id INT AUTO_INCREMENT PRIMARY KEY,
    _steamID VARCHAR(20),
    _name VARCHAR(70),
    _description TEXT,
    _model VARCHAR(128),
    _faction INT,
    _class INT DEFAULT 0,
    _money INT DEFAULT 0,
    _data TEXT,  -- JSON encoded character data
    _attribs TEXT,  -- JSON encoded attributes
    _createTime INT,
    _lastJoinTime INT,
    FOREIGN KEY (_steamID) REFERENCES ix_players(_steamID)
);

-- Inventories table
CREATE TABLE ix_inventories (
    _id INT AUTO_INCREMENT PRIMARY KEY,
    _character INT,
    _inventory TEXT,  -- JSON encoded inventory data
    FOREIGN KEY (_character) REFERENCES ix_characters(_id)
);

-- Configuration table
CREATE TABLE ix_config (
    _key VARCHAR(50) PRIMARY KEY,
    _value TEXT
);

-- Plugin-specific tables created as needed
```

### Hook System Architecture

```lua
-- Standard Garry's Mod hook system
hook.Add("HookName", "Identifier", function(args...) end)

-- Helix extends this with plugin method caching
-- When a plugin loads, methods are scanned and cached

Plugin Example:
--------------
PLUGIN.name = "Example"

function PLUGIN:PlayerSay(client, text, teamChat)
    -- This method is automatically registered as a hook
    return text
end

-- Internally, Helix does:
ix.plugin.HOOKS_CACHE["PlayerSay"] = ix.plugin.HOOKS_CACHE["PlayerSay"] or {}
table.insert(ix.plugin.HOOKS_CACHE["PlayerSay"], {
    plugin = PLUGIN,
    method = PLUGIN.PlayerSay
})

-- When hook is called:
function ix.plugin.Call(hookName, ...)
    local cache = ix.plugin.HOOKS_CACHE[hookName]
    if cache then
        for _, data in ipairs(cache) do
            local result = data.method(data.plugin, ...)
            if result ~= nil then
                return result
            end
        end
    end
    return hook.Run(hookName, ...)
end
```

### Network Message Flow

```
Server ─────────────────────────────► Client
       │                              │
       │  net.Start("MessageName")    │
       │  net.WriteType(data)         │
       │  net.Send(player)            │
       │                              │
       │                              ▼
       │                   ┌──────────────────┐
       │                   │  net.Receive()   │
       │                   │  registered      │
       │                   └────────┬─────────┘
       │                            │
       │                            ▼
       │                   ┌──────────────────┐
       │                   │  Read data       │
       │                   │  Process         │
       │                   │  Update UI       │
       │                   └──────────────────┘
       │
       │◄──────────────── Response (if needed)
       │
       ▼
┌──────────────────┐
│  net.Receive()   │
│  registered      │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Process         │
│  Update state    │
│  Sync to other   │
│  clients         │
└──────────────────┘
```

## Performance Considerations

### Hook Caching

- Plugin methods are cached in `HOOKS_CACHE` table
- Reduces table lookups on every hook call
- O(1) access to plugin hooks by name

### Network Optimization

- Minimize net message frequency
- Batch updates when possible
- Use delta compression for large data
- Compress repeated data with enums

### Database Optimization

- Prepared statements prevent SQL injection
- Connection pooling for MySQL
- Batch inserts/updates when possible
- Use transactions for multi-query operations
- Index on frequently queried columns (_steamID, _id)

### UI Optimization

- Reuse Derma panels where possible
- Cache color and font objects
- Minimize paint() hook complexity
- Use PaintManual for off-screen rendering
- Throttle Update() methods

### Memory Management

- Character data unloaded on disconnect
- Inventory instances cleaned up properly
- Weak tables for caches
- Periodic garbage collection hints

## Security Considerations

### Client-Server Trust

- Never trust client input
- Validate all commands server-side
- Check permissions on every action
- Sanitize strings before database insertion
- Use prepared statements

### Permission System

- CAMI integration for admin systems
- Flag-based permissions (single characters)
- Command-level permission checks
- Entity-level access control

### Data Validation

```lua
-- Type system enforces argument types
ix.command.Add("GiveMoney", {
    arguments = {
        ix.type.player,     -- Must be valid player
        ix.type.number      -- Must be number
    },
    OnRun = function(self, client, target, amount)
        -- Arguments are guaranteed to be correct types
        if amount < 0 then return "Amount must be positive" end
        if not client:IsAdmin() then return "No permission" end

        local character = target:GetCharacter()
        if character then
            character:GiveMoney(amount)
        end
    end
})
```

## Extension Points

### Adding New Systems

1. Create library in `gamemode/core/libs/`
2. Follow naming convention: `sh_systemname.lua`
3. Create namespace: `ix.systemname = ix.systemname or {}`
4. Implement functions
5. Add to shared.lua include list

### Adding New Meta Methods

1. Create file in `gamemode/core/meta/`
2. Get meta table: `local meta = FindMetaTable("Player")`
3. Add methods: `function meta:MethodName() end`
4. Document with LDoc comments

### Adding New UI Components

1. Create file in `gamemode/core/derma/`
2. Define panel: `local PANEL = {}`
3. Implement Paint, Think, Init methods
4. Register: `vgui.Register("PanelName", PANEL, "DPanel")`

### Creating Plugins

1. Create plugin folder in `plugins/`
2. Create `sh_plugin.lua` with metadata
3. Add PLUGIN methods for hooks
4. Organize features in subdirectories
5. Use PLUGIN:SetData() for persistent data

## See Also

- [Plugin System](plugins/plugin-system.md)
- [Database System](libraries/database.md)
- [Character System](systems/character.md)
- [Item System](systems/items.md)
- [Best Practices](best-practices.md)
