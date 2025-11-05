# Quick Start Guide

Get up and running with Helix development in minutes.

## Prerequisites

- Garry's Mod installed
- Basic Lua knowledge
- Text editor (VS Code, Sublime Text, Atom, etc.)
- Local dedicated server or test server

## Installation

### 1. Install Helix Framework

Download Helix from GitHub and place in your gamemodes folder:

```
garrysmod/
└── gamemodes/
    └── helix/           # Helix framework
```

### 2. Create Your Schema

**Option A: Use Skeleton Schema (Recommended)**

Download the skeleton schema from https://github.com/nebulouscloud/helix-skeleton

```
garrysmod/
└── gamemodes/
    ├── helix/          # Framework
    └── myschema/       # Your schema
        ├── gamemode/
        │   ├── init.lua
        │   ├── cl_init.lua
        │   └── shared.lua
        └── schema/
            └── sh_schema.lua
```

**Option B: Start from Scratch**

Create a new folder in `gamemodes/` (e.g., `myschema`):

**gamemode/init.lua:**
```lua
AddCSLuaFile("cl_init.lua")
DeriveGamemode("helix")
```

**gamemode/cl_init.lua:**
```lua
DeriveGamemode("helix")
```

**schema/sh_schema.lua:**
```lua
Schema.name = "My Schema"
Schema.author = "Your Name"
Schema.description = "My awesome roleplay schema"

-- Your schema code here
```

### 3. Set Your Gamemode

In your server's startup command or configuration:

```bash
+gamemode "myschema"
```

Or in `server.cfg`:
```
gamemode "myschema"
```

### 4. Start Your Server

Launch your server and join. You should see the Helix character selection screen!

## Your First Plugin

Let's create a simple plugin that welcomes players.

### 1. Create Plugin Folder

```
myschema/
└── plugins/
    └── welcome/
        └── sh_plugin.lua
```

### 2. Create Plugin File

**plugins/welcome/sh_plugin.lua:**

```lua
PLUGIN.name = "Welcome Plugin"
PLUGIN.author = "Your Name"
PLUGIN.description = "Welcomes players when they join"

function PLUGIN:PlayerLoadedCharacter(client, character)
    -- Send welcome message
    timer.Simple(1, function()
        if IsValid(client) then
            client:ChatPrint("Welcome, " .. character:GetName() .. "!")
            client:ChatPrint("This is your schema!")
        end
    end)
end
```

### 3. Reload

Type `sh_rebuildschema` in server console or restart your server.

### 4. Test

Join with a character and you should see the welcome message!

## Your First Item

Let's create a health kit item.

### 1. Create Items Folder

```
myschema/
└── schema/
    └── items/
        └── sh_healthkit.lua
```

### 2. Define Item

**schema/items/sh_healthkit.lua:**

```lua
ITEM.name = "Health Kit"
ITEM.description = "Restores 50 health"
ITEM.model = "models/items/healthkit.mdl"
ITEM.width = 2
ITEM.height = 1
ITEM.category = "Medical"
ITEM.price = 100

ITEM.functions.Use = {
    name = "Use",
    icon = "icon16/heart.png",
    OnRun = function(item)
        local client = item.player

        if SERVER then
            local health = client:Health()
            local maxHealth = client:GetMaxHealth()

            if health >= maxHealth then
                client:NotifyLocalized("You are already at full health")
                return false
            end

            client:SetHealth(math.min(health + 50, maxHealth))
            client:EmitSound("items/medshot4.wav")
        end

        return true  -- Remove item after use
    end,
    OnCanRun = function(item)
        return item.player:Health() < item.player:GetMaxHealth()
    end
}
```

### 3. Give Yourself the Item

In console (as admin):
```
ix_giveitem (your name) sh_healthkit
```

Or via command:
```
/giveitem "Your Name" sh_healthkit
```

### 4. Test

Open your inventory (Tab by default) and use the health kit!

## Your First Command

Let's create a command to check server time.

### 1. Add Command to Plugin

**plugins/welcome/sh_plugin.lua:**

```lua
PLUGIN.name = "Welcome Plugin"
PLUGIN.author = "Your Name"
PLUGIN.description = "Welcomes players and provides utilities"

-- Welcome message from before
function PLUGIN:PlayerLoadedCharacter(client, character)
    timer.Simple(1, function()
        if IsValid(client) then
            client:ChatPrint("Welcome, " .. character:GetName() .. "!")
            client:ChatPrint("Type /time to see server time!")
        end
    end)
end

-- Add command
ix.command.Add("Time", {
    description = "Shows the current server time",
    OnRun = function(self, client)
        return "Server time: " .. os.date("%H:%M:%S")
    end
})
```

### 2. Reload and Test

Type `/time` in chat to see the server time!

## Your First Faction

Let's create a citizen faction.

### 1. Create Factions Folder

```
myschema/
└── schema/
    └── factions/
        └── sh_citizen.lua
```

### 2. Define Faction

**schema/factions/sh_citizen.lua:**

```lua
FACTION.name = "Citizen"
FACTION.description = "Regular city residents"
FACTION.color = Color(50, 150, 50)
FACTION.isDefault = true  -- Default faction for new characters

FACTION.models = {
    "models/player/group01/male_01.mdl",
    "models/player/group01/male_02.mdl",
    "models/player/group01/male_03.mdl",
    "models/player/group01/male_04.mdl",
    "models/player/group01/female_01.mdl",
    "models/player/group01/female_02.mdl",
    "models/player/group01/female_03.mdl",
    "models/player/group01/female_04.mdl"
}

function FACTION:OnCharacterCreated(client, character)
    -- Give starting items
    local inventory = character:GetInventory()
    inventory:Add("sh_healthkit")

    -- Give starting money
    character:GiveMoney(100)
end

FACTION_CITIZEN = FACTION.index
```

### 3. Create Character

Restart server and create a new character. You should see "Citizen" as a faction option!

## Common Development Tasks

### Reloading

**Reload schema/plugins:**
```
sh_rebuildschema
```

**Reload just config:**
```lua
ix.config.Load()
```

**Restart server completely:**
```
changelevel gm_construct  # Or your map
```

### Giving Items

```
/giveitem "Player Name" item_uniqueid
```

### Setting Money

```
/setmoney "Player Name" 1000
```

### Checking Variables

```lua
lua_run_cl PrintTable(ix.config.stored)  # View all config
lua_run_cl PrintTable(ix.item.list)      # View all items
lua_run PrintTable(ix.faction.indices)   # View all factions
```

### Debugging

Enable developer mode:
```
developer 1
```

View errors:
```
con_filter_enable 1
con_filter_text "error"
```

## File Structure Reference

```
myschema/
├── gamemode/
│   ├── init.lua              # Server entry point
│   ├── cl_init.lua           # Client entry point
│   └── shared.lua            # Shared initialization (optional)
├── schema/
│   ├── sh_schema.lua         # Schema definition (required)
│   ├── sv_schema.lua         # Server schema code
│   ├── cl_schema.lua         # Client schema code
│   ├── items/                # Item definitions
│   │   ├── sh_item1.lua
│   │   └── sh_item2.lua
│   ├── factions/             # Faction definitions
│   │   ├── sh_citizen.lua
│   │   └── sh_police.lua
│   ├── classes/              # Class definitions
│   │   └── sh_medic.lua
│   └── languages/            # Localizations
│       └── sh_english.lua
└── plugins/                  # Your custom plugins
    ├── welcome/
    │   └── sh_plugin.lua
    └── customfeature/
        ├── sh_plugin.lua
        ├── sv_hooks.lua
        └── cl_hooks.lua
```

## Realm Separation

Always check which realm code runs on:

```lua
if SERVER then
    -- Server-only code
    print("This runs on server")
end

if CLIENT then
    -- Client-only code
    print("This runs on client")
end

-- Shared code (runs on both)
print("This runs everywhere")
```

## Common Patterns

### Getting Player's Character

```lua
local character = client:GetCharacter()
if not character then
    -- Player has no active character
    return
end

-- Use character
local name = character:GetName()
local money = character:GetMoney()
```

### Giving Items to Player

```lua
local character = client:GetCharacter()
if character then
    local inventory = character:GetInventory()
    inventory:Add("sh_healthkit")
end
```

### Creating Custom Data

```lua
-- Set data
character:SetData("reputation", 100)
character:SetData("quests", {})

-- Get data
local reputation = character:GetData("reputation", 0)
local quests = character:GetData("quests", {})
```

### Sending Messages

```lua
-- Chat message
client:ChatPrint("Hello!")

-- Notification
client:Notify("Quest completed!")

-- Localized message
client:NotifyLocalized("questCompleted", "Quest Name")
```

## Next Steps

Now that you have the basics:

1. **Explore the base plugins** - See `helix/plugins/` for examples
2. **Read the documentation** - Check `docs/` folder
3. **Study the HL2 RP schema** - Great example at https://github.com/nebulouscloud/helix-hl2rp
4. **Join the community** - Discord: https://discord.gg/2AutUcF
5. **Check the plugin center** - https://plugins.gethelix.co

## Useful Resources

### Documentation

- [Architecture Overview](../architecture.md)
- [Plugin System](../plugins/plugin-system.md)
- [Item System](../systems/items.md)
- [Command System](../systems/commands.md)
- [Character System](../systems/character.md)

### Examples

- Helix HL2 RP Schema: https://github.com/nebulouscloud/helix-hl2rp
- Helix Skeleton: https://github.com/nebulouscloud/helix-skeleton
- Plugin Examples: https://github.com/nebulouscloud/helix-plugins

### Community

- Official Website: https://gethelix.co
- Documentation: https://docs.gethelix.co
- Discord: https://discord.gg/2AutUcF
- GitHub: https://github.com/nebulouscloud/helix

## Troubleshooting

### "attempt to index nil value"

**Cause**: Trying to access a character that doesn't exist.

**Fix**:
```lua
-- Bad
local money = client:GetCharacter():GetMoney()

-- Good
local character = client:GetCharacter()
if character then
    local money = character:GetMoney()
end
```

### "no such file or directory"

**Cause**: File path is incorrect.

**Fix**: Check your file paths and naming conventions. Use `sh_` prefix for shared files.

### Items not appearing

**Cause**: Items need to be in `items/` folder and properly named.

**Fix**:
1. Place in correct folder: `schema/items/` or `plugins/yourplugin/items/`
2. Name with `sh_` prefix: `sh_itemname.lua`
3. Restart server or run `sh_rebuildschema`

### Commands not working

**Cause**: Command wasn't registered properly.

**Fix**:
1. Use `ix.command.Add()` not `concommand.Add()`
2. Place in plugin or schema file
3. Reload with `sh_rebuildschema`

### Characters not saving

**Cause**: Database not configured or connection failed.

**Fix**:
1. Check server console for database errors
2. Verify `helix.yml` configuration if using MySQL
3. Make sure SQLite file has write permissions

## Quick Reference Card

```lua
-- Get character
local char = client:GetCharacter()

-- Character data
char:GetName()
char:GetMoney()
char:GetInventory()
char:GetFaction()
char:SetData(key, value)
char:GetData(key, default)

-- Give/take money
char:GiveMoney(amount)
char:TakeMoney(amount)
char:HasMoney(amount)

-- Inventory
local inv = char:GetInventory()
inv:Add(itemID)
inv:Remove(itemID)
inv:GetItems()

-- Messages
client:ChatPrint("text")
client:Notify("text")

-- Commands
ix.command.Add("Name", {
    description = "...",
    OnRun = function(self, client)
        return "result"
    end
})

-- Items
ITEM.name = "Name"
ITEM.description = "Description"
ITEM.model = "models/..."
ITEM.functions.Use = {
    OnRun = function(item)
        -- code
        return true  -- remove item
    end
}

-- Plugin hooks
function PLUGIN:HookName(args)
    -- code
end
```

Happy developing! 🎮
