# Development Environment Setup

> **Reference**: For deployment configuration, see `helix.example.yml` and `docs/manual/getting-started.md`

Complete guide to setting up a professional Helix development environment for creating schemas and plugins.

## ⚠️ Important: Use Helix's Built-in Tools

**Always use Helix's included file loading system** rather than creating custom loading mechanisms. The framework provides:
- Automatic file inclusion via `ix.util.Include()` and `ix.util.IncludeDir()`
- Built-in hot-reloading with `sh_rebuildschema`
- Proper realm separation (server/client/shared)

## Prerequisites

Before starting Helix development, ensure you have:

### Required
- **Garry's Mod** - Latest version installed via Steam
- **Basic Lua knowledge** - Understanding of variables, functions, tables
- **Text Editor** - See IDE Setup section below
- **Git** - Version control for your code

### Recommended
- **Local Dedicated Server** - For testing without hosting costs
- **Basic SQL knowledge** - For database operations
- **Garry's Mod SDK** - For model/material work

### Optional
- **MySQL Server** - For production database (not needed for development)
- **Image Editor** - For custom icons/materials
- **FTP Client** - For remote server deployment

## Quick Start (5 Minutes)

### 1. Install Helix Framework

Download Helix from GitHub:
```bash
cd "C:/Program Files (x86)/Steam/steamapps/common/GarrysMod/garrysmod/gamemodes"
git clone https://github.com/NebulousCloud/helix.git helix
```

Or download ZIP and extract to: `garrysmod/gamemodes/helix/`

### 2. Create Development Schema

**Option A: Use Skeleton (Recommended)**
```bash
cd garrysmod/gamemodes
git clone https://github.com/NebulousCloud/helix-skeleton.git myschema
cd myschema
mv skeleton.txt myschema.txt
```

Edit `myschema.txt`:
```
"myschema"
{
    "base"      "helix"
    "title"     "My Schema"
    "author"    "Your Name"
    "version"   "1.0"
}
```

**Option B: Start from Scratch**

See the [Creating from Scratch](#creating-from-scratch) section below.

### 3. Start Development Server

Create a shortcut with target:
```
"C:\...\steamapps\common\GarrysMod\srcds.exe" -console -game garrysmod +gamemode myschema +map gm_construct +maxplayers 4
```

Or on Linux:
```bash
./srcds_run -game garrysmod +gamemode myschema +map gm_construct +maxplayers 4
```

### 4. Join and Test

1. Start your server via shortcut
2. Open Garry's Mod
3. Open console (~) and type: `connect localhost`
4. You should see the Helix character selection screen!

## IDE Setup

### Visual Studio Code (Recommended)

**Install Extensions:**
```
1. GLua Enhanced (WilliamVenner.glua-enhanced)
2. Lua (sumneko.lua)
3. Helix Snippets (search "helix" or "nutscript")
4. GitLens (for version control)
```

**Configure GLua Settings:**

Create `.vscode/settings.json` in your schema folder:
```json
{
    "glua-enhanced.includePaths": [
        "${workspaceFolder}/../helix/gamemode",
        "${workspaceFolder}/../helix/plugins"
    ],
    "glua-enhanced.autocompleteLibraries": true,
    "files.associations": {
        "*.lua": "lua"
    },
    "editor.tabSize": 4,
    "editor.insertSpaces": false,
    "editor.detectIndentation": false
}
```

**Install Lua Language Server:**

Create `.luarc.json` in schema root:
```json
{
    "runtime": {
        "version": "LuaJIT"
    },
    "workspace": {
        "library": [
            "${3rd}/luabit/library",
            "../helix/gamemode/core"
        ],
        "checkThirdParty": false
    },
    "diagnostics": {
        "globals": [
            "SERVER",
            "CLIENT",
            "PLUGIN",
            "ITEM",
            "FACTION",
            "CLASS",
            "Schema",
            "ix",
            "hook",
            "timer",
            "net",
            "player",
            "team"
        ]
    }
}
```

### Sublime Text

**Install Packages** (via Package Control):
- GLua
- SublimeLinter-glualint
- GitGutter

**User Settings:**
```json
{
    "tab_size": 4,
    "translate_tabs_to_spaces": false,
    "detect_indentation": false
}
```

### Atom

**Install Packages:**
```
language-glua
autocomplete-glua
```

## Project Structure

### Standard Directory Layout

```
garrysmod/
├── gamemodes/
│   ├── helix/                  # Framework (don't modify)
│   │   ├── gamemode/
│   │   │   ├── core/           # Core libraries
│   │   │   ├── derma/          # UI components
│   │   │   ├── config/         # Default config
│   │   │   └── languages/      # Localizations
│   │   ├── plugins/            # Base plugins
│   │   └── items/              # Base items
│   │
│   └── myschema/               # Your schema
│       ├── gamemode/           # Bootstrap files
│       │   ├── init.lua        # Server entry
│       │   ├── cl_init.lua     # Client entry
│       │   └── shared.lua      # Shared (optional)
│       ├── schema/             # Your schema code
│       │   ├── sh_schema.lua   # Schema definition
│       │   ├── sv_schema.lua   # Server code
│       │   ├── cl_schema.lua   # Client code
│       │   ├── items/          # Custom items
│       │   ├── factions/       # Factions
│       │   ├── classes/        # Classes
│       │   ├── derma/          # UI customization
│       │   ├── hooks/          # Hook files
│       │   └── languages/      # Translations
│       └── plugins/            # Your plugins
│           └── myplugin/
│               ├── sh_plugin.lua
│               ├── sv_hooks.lua
│               ├── cl_hooks.lua
│               └── items/
└── data/
    └── helix/                  # Runtime data
        └── helix.db            # SQLite database (auto-created)
```

### File Naming Conventions

**Always follow these conventions:**

- `sh_` - Shared (runs on both server and client)
- `sv_` - Server only
- `cl_` - Client only

```lua
// ✅ CORRECT
schema/sh_schema.lua          // Main schema file (shared)
schema/sv_hooks.lua           // Server-side hooks
schema/cl_hooks.lua           // Client-side hooks
plugins/myplugin/sh_plugin.lua  // Plugin definition
items/sh_medkit.lua           // Item definition

// ❌ WRONG
schema/schema.lua             // Missing realm prefix
schema/server.lua             // Use sv_ prefix
schema/client_hooks.lua       // Use cl_ not client_
```

## Creating From Scratch

### Step 1: Create Gamemode Folder

```bash
cd garrysmod/gamemodes
mkdir myschema
cd myschema
mkdir gamemode
mkdir schema
mkdir plugins
```

### Step 2: Create Bootstrap Files

**gamemode/init.lua:**
```lua
-- Server entry point
AddCSLuaFile("cl_init.lua")
AddCSLuaFile("shared.lua")

include("shared.lua")
DeriveGamemode("helix")
```

**gamemode/cl_init.lua:**
```lua
-- Client entry point
include("shared.lua")
DeriveGamemode("helix")
```

**gamemode/shared.lua** (optional):
```lua
-- Shared initialization
GM.Name = "My Schema"
GM.Author = "Your Name"
GM.Email = "N/A"
GM.Website = "N/A"

-- Any shared pre-initialization code here
```

### Step 3: Create Schema Definition

**schema/sh_schema.lua:**
```lua
Schema.name = "My Schema"
Schema.author = "Your Name"
Schema.description = "A roleplay schema built on Helix"

-- Include additional schema files
ix.util.Include("sv_schema.lua")
ix.util.Include("cl_schema.lua")

-- Load schema directories
ix.util.IncludeDir("schema/hooks")
ix.util.IncludeDir("schema/derma")

-- Schema-specific functions
function Schema:GetGameDescription()
    return self.name
end

-- Configuration defaults
ix.config.Set("color", Color(50, 150, 50))
ix.config.Set("font", "Arial")
```

**schema/sv_schema.lua:**
```lua
-- Server-side schema code
function Schema:PlayerInitialSpawn(client)
    print(client:Name() .. " connected to " .. self.name)
end
```

**schema/cl_schema.lua:**
```lua
-- Client-side schema code
function Schema:HUDPaint()
    -- Custom HUD elements
end
```

### Step 4: Create Gamemode Info File

**myschema.txt:**
```
"myschema"
{
    "base"          "helix"
    "title"         "My Schema"
    "author"        "Your Name"
    "version"       "1.0"
    "website"       "https://yourwebsite.com"
    "menusystem"    "1"
}
```

## Development Workflow

### Starting Your Server

**Windows Batch Script** (save as `start_server.bat`):
```batch
@echo off
cd "C:\Program Files (x86)\Steam\steamapps\common\GarrysMod"
start srcds.exe -console -game garrysmod +gamemode myschema +map gm_construct +maxplayers 4 +hostname "Dev Server" +sv_lan 1
```

**Linux Shell Script** (save as `start_server.sh`):
```bash
#!/bin/bash
cd ~/gmod
./srcds_run -console -game garrysmod \
    +gamemode myschema \
    +map gm_construct \
    +maxplayers 4 \
    +hostname "Dev Server" \
    +sv_lan 1
```

### Development Server Config

**server.cfg** (in `garrysmod/cfg/`):
```
// Development settings
hostname "My Schema Dev Server"
sv_lan 1
sv_password ""
maxplayers 4

// Development helpers
sbox_godmode 0
sbox_noclip 1
sbox_maxprops 1000

// Developer console
developer 1
con_logfile "console.log"

// Error logging
lua_log_sv 1
lua_log_cl 1

// Exec gamemode
exec banned_user.cfg
exec banned_ip.cfg
```

### Hot Reloading

**Reload schema/plugins without restart:**
```
// In server console
sh_rebuildschema

// Or via chat command (admin)
/sh_rebuildschema
```

**⚠️ Note:** This reloads Lua files but NOT:
- Entities
- Effects
- Weapons
- Maps
- Models
- Materials

For those, you must restart the server.

### Testing Workflow

**1. Make Code Changes**
```lua
// Edit files in schema/ or plugins/
```

**2. Reload Schema**
```
sh_rebuildschema
```

**3. Test Changes**
```
// Join server and test
// Check console for errors
```

**4. Check Logs**
```
// Server console for errors
// Client console: Con_LogFile "client.log"
```

## Database Setup

### SQLite (Development)

**Default configuration** - no setup required!

SQLite database is automatically created at:
```
garrysmod/data/helix/helix.db
```

**⚠️ Important**: This file is binary. Don't edit manually. Use SQL queries via `ix.db` functions.

### MySQL (Production)

**1. Install MySQLOO Module**

Download from: https://github.com/FredyH/MySQLOO/releases

Place in: `garrysmod/lua/bin/`
- Windows: `gmsv_mysqloo_win32.dll`
- Linux: `gmsv_mysqloo_linux.dll`

**2. Create Configuration**

Copy `helix.example.yml` to `helix.yml` in `gamemodes/helix/`:

```yaml
database:
  adapter: "mysqloo"
  hostname: "localhost"
  username: "helix_user"
  password: "your_password"
  database: "helix_db"
  port: 3306
```

**⚠️ Critical:** Use exactly 2 spaces for indentation, NOT tabs!

**3. Create Database**

```sql
CREATE DATABASE helix_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'helix_user'@'localhost' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON helix_db.* TO 'helix_user'@'localhost';
FLUSH PRIVILEGES;
```

**4. Test Connection**

Start server and check console:
```
[Helix] Connected to MySQL database 'helix_db'
```

## Version Control Setup

### Initialize Git Repository

```bash
cd garrysmod/gamemodes/myschema
git init
```

### Create .gitignore

**.gitignore:**
```gitignore
# Database
helix.yml
*.db
*.db-journal

# Logs
*.log
console.log

# OS Files
.DS_Store
Thumbs.db

# IDE
.vscode/
.idea/
*.sublime-workspace

# Temp files
*.tmp
*.bak
*~

# Build artifacts
*.gma
*.cache
```

### Initial Commit

```bash
git add .
git commit -m "Initial schema setup"
```

### Create Repository

On GitHub/GitLab:
1. Create new repository
2. Add remote: `git remote add origin https://github.com/username/myschema.git`
3. Push: `git push -u origin master`

## Development Tools

### Console Commands

**Essential commands:**
```
// Reload schema
sh_rebuildschema

// Give yourself admin
sh_makeadmin

// Set developer mode
developer 1

// No clip
noclip

// God mode
god

// Give item
ix_giveitem "Player Name" item_uniqueid

// Set money
/setmoney "Player Name" 1000

// Spawn as NPC
/becomeclasss class_uniqueid
```

### Useful Binds

Add to `autoexec.cfg`:
```
// Quick reload
bind "F5" "sh_rebuildschema"

// No clip toggle
bind "F6" "noclip"

// Open inventory
bind "TAB" "+ix_menu"

// Quick console
bind "F1" "toggleconsole"
```

### Debug Helpers

**Enable error display:**
```lua
-- In console
lua_openscript_cl your_file.lua  // Client
lua_openscript your_file.lua     // Server
```

**Print variable:**
```lua
PrintTable(ix.faction.indices)  // Print faction table
print(tostring(value))           // Print single value
```

## Common Development Tasks

### Creating New Plugin

```bash
cd myschema/plugins
mkdir myplugin
cd myplugin
touch sh_plugin.lua
```

**sh_plugin.lua:**
```lua
PLUGIN.name = "My Plugin"
PLUGIN.author = "Your Name"
PLUGIN.description = "Does something cool"

function PLUGIN:PlayerLoadedCharacter(client, character)
    client:Notify("Plugin is working!")
end
```

### Creating New Item

```bash
cd myschema/schema/items
touch sh_myitem.lua
```

**sh_myitem.lua:**
```lua
ITEM.name = "My Item"
ITEM.description = "An example item"
ITEM.model = "models/props_junk/cardboard_box001a.mdl"
ITEM.width = 1
ITEM.height = 1
```

### Creating New Faction

```bash
cd myschema/schema/factions
touch sh_myfaction.lua
```

**sh_myfaction.lua:**
```lua
FACTION.name = "My Faction"
FACTION.description = "Faction description"
FACTION.color = Color(100, 150, 200)
FACTION.isDefault = false

FACTION.models = {
    "models/player/group01/male_01.mdl"
}

FACTION_MYFACTION = FACTION.index
```

## Troubleshooting Setup

### Server won't start

**Symptoms**: Server crashes immediately or shows errors

**Solutions**:
1. Check gamemode name matches folder name
2. Verify `DeriveGamemode("helix")` is in init files
3. Ensure Helix is installed in `gamemodes/helix/`
4. Check server console for specific error

### "Attempt to call nil value"

**Cause**: Trying to use Helix function before framework loads

**Fix**:
```lua
// ❌ WRONG - Runs too early
ix.command.Add("test", {...})  // At top of file

// ✅ CORRECT - Use hook
function PLUGIN:InitializedPlugins()
    ix.command.Add("test", {...})
end
```

### "No such file or directory"

**Cause**: File path or naming is wrong

**Solutions**:
1. Check file naming conventions (sh_, sv_, cl_)
2. Verify path case sensitivity (Linux)
3. Ensure file is in correct folder
4. Check for typos in Include() calls

### Database connection fails

**Cause**: MySQL configuration incorrect

**Solutions**:
1. Verify MySQLOO is installed
2. Check helix.yml indentation (2 spaces!)
3. Test MySQL credentials with MySQL client
4. Check server console for specific error
5. Ensure MySQL server is running

### Changes not appearing

**Cause**: Files not reloading properly

**Solutions**:
1. Run `sh_rebuildschema`
2. Rejoin server
3. Full server restart if adding entities/weapons
4. Check console for Lua errors preventing reload

## Performance Optimization

### Development Mode

For faster testing:
```lua
// sv_schema.lua
ix.config.Set("ticketTime", 0)        // Skip intro
ix.config.Set("saveInterval", 300)    // Save less often
ix.config.Set("maxAttributes", 100)   // Faster attribute gains
```

### Profiling

Find slow code:
```lua
local start = SysTime()
-- Your code here
print("Took: " .. (SysTime() - start) .. " seconds")
```

## Best Practices

### ✅ DO

- Use version control (Git)
- Test on dedicated server, not listen server
- Follow Helix file naming conventions
- Comment your code thoroughly
- Use ix.util.Include() for file loading
- Keep backup of working versions
- Test with multiple players

### ❌ DON'T

- Modify Helix core files directly
- Commit database files or passwords
- Test only in single player
- Skip error checking
- Hardcode file paths
- Work without backups
- Deploy untested code to production

## Next Steps

Now that your environment is set up:

1. **Read Framework Documentation**
   - [Architecture Overview](../architecture.md)
   - [Plugin System](../plugins/plugin-system.md)
   - [Character System](../systems/character.md)

2. **Create Your First Plugin**
   - [First Plugin Tutorial](first-plugin.md)
   - [Plugin Best Practices](../plugins/best-practices.md)

3. **Study Examples**
   - Base plugins in `helix/plugins/`
   - HL2 RP Schema: https://github.com/nebulouscloud/helix-hl2rp
   - Skeleton Schema: https://github.com/nebulouscloud/helix-skeleton

4. **Join Community**
   - Discord: https://discord.gg/2AutUcF
   - Forums: https://forums.gethelix.co
   - Plugin Center: https://plugins.gethelix.co

## See Also

- [Quick Start Guide](quick-start.md) - Fast introduction
- [First Plugin Tutorial](first-plugin.md) - Create your first plugin
- [Getting Started Manual](../manual/getting-started.md) - Official installation guide
- [Debugging Guide](debugging.md) - Troubleshooting techniques
