# Creating Your First Plugin

> **Reference**: `gamemode/core/libs/sh_plugin.lua`

This tutorial will guide you through creating a complete, working plugin from scratch. You'll learn how to use Helix's plugin system to add custom functionality without modifying core files.

## ⚠️ Important: Use Helix Plugin System

**Always use Helix's plugin system** rather than modifying core files. The framework provides:
- Automatic file loading and organization
- Built-in data persistence with `PLUGIN:SetData()` and `PLUGIN:GetData()`
- Hook system integration - any function you add to `PLUGIN` automatically becomes a hook
- Safe unloading and reloading
- Proper realm (client/server) handling

## What You'll Build

In this tutorial, you'll create a "Welcome Message" plugin that:
- Shows a message when players join the server
- Tracks how many times each player has joined
- Provides an admin command to check join counts
- Saves data persistently to disk

## Step 1: Create Plugin Directory

Navigate to your schema's plugin directory and create a new folder:

```
yourschema/plugins/welcomemessage/
```

**⚠️ Important**: Plugin folder names should be:
- All lowercase
- No spaces (use underscores if needed)
- Descriptive and unique

## Step 2: Create Main Plugin File

Create `sh_plugin.lua` in your plugin folder. This is the **only required file** for a plugin.

**File**: `plugins/welcomemessage/sh_plugin.lua`

```lua
-- sh_ prefix means this file runs on both client and server
PLUGIN.name = "Welcome Message"
PLUGIN.author = "Your Name"
PLUGIN.description = "Displays welcome messages and tracks player joins"

-- This runs when a player first connects (before character selection)
function PLUGIN:PlayerInitialSpawn(client)
    -- SERVER check because we only want this to run on server
    if SERVER then
        -- Get player's unique ID
        local steamID = client:SteamID()

        -- Load plugin data (automatically persistent)
        local data = self:GetData()
        data.joinCounts = data.joinCounts or {}

        -- Increment join count
        local count = (data.joinCounts[steamID] or 0) + 1
        data.joinCounts[steamID] = count

        -- Save data (Helix handles disk I/O automatically)
        self:SetData(data)

        -- Send message to player
        timer.Simple(1, function()
            if IsValid(client) then
                client:ChatPrint("Welcome to the server, " .. client:Name() .. "!")
                client:ChatPrint("You have joined " .. count .. " times.")
            end
        end)
    end
end
```

**That's it!** You now have a working plugin. Let's break down what's happening:

### Plugin Metadata

**Reference**: `gamemode/core/libs/sh_plugin.lua:23-24`

```lua
PLUGIN.name = "Welcome Message"      -- Display name in plugin list
PLUGIN.author = "Your Name"          -- Your name or organization
PLUGIN.description = "..."           -- What your plugin does
```

These fields are **required** for every plugin. The framework reads these when loading your plugin.

### Automatic Hook Registration

**Reference**: `gamemode/core/libs/sh_plugin.lua:78-83`

When you create a function on `PLUGIN`, Helix automatically registers it as a hook:

```lua
function PLUGIN:PlayerInitialSpawn(client)
    -- This automatically hooks into the PlayerInitialSpawn event
    -- You don't need to call hook.Add() manually!
end
```

### Persistent Data Storage

**Reference**: `gamemode/core/libs/sh_plugin.lua:64-70`

```lua
-- Load data
local data = self:GetData()  -- Returns saved table or {}

-- Modify data
data.joinCounts = data.joinCounts or {}
data.joinCounts[steamID] = count

-- Save data (automatic file I/O)
self:SetData(data)
```

The framework automatically:
- Creates a data file for your plugin
- Loads it when the plugin initializes
- Saves it when you call `SetData()` or server shuts down
- Handles JSON encoding/decoding

## Step 3: Testing Your Plugin

1. Place your plugin folder in `yourschema/plugins/welcomemessage/`
2. Restart your server or reload plugins
3. Connect to the server
4. You should see welcome messages in chat!

## Step 4: Add a Command

Let's add an admin command to check join counts. Create a new file for organization.

**File**: `plugins/welcomemessage/sh_commands.lua`

```lua
-- Register a chat command
ix.command.Add("CheckJoins", {
    description = "Check how many times a player has joined",
    adminOnly = true,  -- Only admins can use this
    arguments = {
        ix.type.player  -- First argument: target player
    },
    OnRun = function(self, client, target)
        -- Access plugin data
        local data = PLUGIN:GetData()
        local count = data.joinCounts[target:SteamID()] or 0

        return target:Name() .. " has joined " .. count .. " times."
    end
})
```

### Loading Additional Files

**Reference**: `gamemode/core/libs/sh_plugin.lua:55`

Now update `sh_plugin.lua` to load this file:

```lua
PLUGIN.name = "Welcome Message"
PLUGIN.author = "Your Name"
PLUGIN.description = "Displays welcome messages and tracks player joins"

-- Load command file (automatically handles realm)
ix.util.Include("sh_commands.lua")

function PLUGIN:PlayerInitialSpawn(client)
    -- ... existing code ...
end
```

The `ix.util.Include()` function:
- Automatically handles file paths relative to your plugin
- Respects realm prefixes (`sv_`, `cl_`, `sh_`)
- Adds the file to client downloads if needed

## Step 5: Add Client-Side Features

Let's add a notification sound on the client.

**File**: `plugins/welcomemessage/cl_hooks.lua`

```lua
-- This file runs CLIENT-ONLY (cl_ prefix)

-- Network message receiver
net.Receive("WelcomeMessageSound", function()
    surface.PlaySound("buttons/button14.wav")
end)

-- Visual notification
function PLUGIN:InitializedPlugins()
    -- This runs after all plugins load
    print("[Welcome Message] Plugin loaded on client!")
end
```

**File**: `plugins/welcomemessage/sv_hooks.lua`

```lua
-- This file runs SERVER-ONLY (sv_ prefix)

-- Register network message
util.AddNetworkString("WelcomeMessageSound")

function PLUGIN:PlayerInitialSpawn(client)
    local steamID = client:SteamID()
    local data = self:GetData()
    data.joinCounts = data.joinCounts or {}
    local count = (data.joinCounts[steamID] or 0) + 1
    data.joinCounts[steamID] = count
    self:SetData(data)

    timer.Simple(1, function()
        if IsValid(client) then
            client:ChatPrint("Welcome to the server, " .. client:Name() .. "!")
            client:ChatPrint("You have joined " .. count .. " times.")

            -- Send network message to play sound
            net.Start("WelcomeMessageSound")
            net.Send(client)
        end
    end)
end
```

Update `sh_plugin.lua` to load these files:

```lua
PLUGIN.name = "Welcome Message"
PLUGIN.author = "Your Name"
PLUGIN.description = "Displays welcome messages and tracks player joins"

ix.util.Include("sh_commands.lua")
ix.util.Include("sv_hooks.lua")
ix.util.Include("cl_hooks.lua")

-- Main plugin code can stay here or be moved to sv_hooks.lua
```

## Complete Plugin Structure

Your final plugin structure should look like:

```
plugins/welcomemessage/
├── sh_plugin.lua          # Main plugin file (required)
├── sh_commands.lua        # Commands (shared)
├── sv_hooks.lua           # Server-only hooks
└── cl_hooks.lua           # Client-only hooks
```

## Complete Working Example

Here's the full, production-ready version:

**`sh_plugin.lua`**:
```lua
PLUGIN.name = "Welcome Message"
PLUGIN.author = "Your Name"
PLUGIN.description = "Displays welcome messages and tracks player joins"
PLUGIN.version = "1.0.0"

-- Load additional files
ix.util.Include("sh_commands.lua")
ix.util.Include("sv_hooks.lua")
ix.util.Include("cl_hooks.lua")

-- Initialize plugin data
function PLUGIN:InitializedPlugins()
    if SERVER then
        -- Ensure data structure exists
        local data = self:GetData()
        if not data.joinCounts then
            data.joinCounts = {}
            self:SetData(data)
        end
    end
end

-- Cleanup when plugin unloads
function PLUGIN:OnPluginUnloaded()
    -- Clean up any timers, panels, etc.
    if SERVER then
        -- Save final data
        self:SetData(self:GetData())
    end
end
```

**`sv_hooks.lua`**:
```lua
util.AddNetworkString("WelcomeMessageSound")

function PLUGIN:PlayerInitialSpawn(client)
    local steamID = client:SteamID()

    -- Get data
    local data = self:GetData()
    data.joinCounts = data.joinCounts or {}

    -- Update count
    local count = (data.joinCounts[steamID] or 0) + 1
    data.joinCounts[steamID] = count

    -- Save
    self:SetData(data)

    -- Notify player
    timer.Simple(1, function()
        if not IsValid(client) then return end

        client:ChatPrint("Welcome to the server, " .. client:Name() .. "!")
        client:ChatPrint("You have joined " .. count .. " times.")

        -- Play sound on client
        net.Start("WelcomeMessageSound")
        net.Send(client)
    end)
end
```

**`cl_hooks.lua`**:
```lua
net.Receive("WelcomeMessageSound", function()
    surface.PlaySound("buttons/button14.wav")
end)
```

**`sh_commands.lua`**:
```lua
ix.command.Add("CheckJoins", {
    description = "Check how many times a player has joined the server",
    adminOnly = true,
    arguments = {
        ix.type.player
    },
    OnRun = function(self, client, target)
        local data = PLUGIN:GetData()
        local count = data.joinCounts[target:SteamID()] or 0

        return target:Name() .. " has joined " .. count .. " times."
    end
})
```

## Best Practices

### ✅ DO

- **Use `self` to access plugin data**: `self:GetData()`, `self.myVariable`
- **Check realm before realm-specific code**: Wrap server code in `if SERVER then`
- **Validate entities**: Always check `IsValid(entity)` before using entities
- **Use timer.Simple for delayed actions**: Gives time for player to fully load
- **Save data after modifications**: Call `self:SetData(data)` after changes
- **Use `ix.util.Include()`**: Let framework handle file loading
- **Access plugin methods with `PLUGIN:`**: In commands, use `PLUGIN:GetData()`

### ❌ DON'T

**Don't create global variables:**
```lua
-- WRONG
joinCounts = {}  -- Pollutes global namespace!

-- RIGHT
PLUGIN.joinCounts = {}  -- Scoped to plugin
-- OR use self:SetData() for persistence
```

**Don't manually call `hook.Add()`:**
```lua
-- WRONG
hook.Add("PlayerInitialSpawn", "MyPlugin", function(client)
    -- ...
end)

-- RIGHT
function PLUGIN:PlayerInitialSpawn(client)
    -- Automatically registered!
end
```

**Don't modify core files:**
```lua
-- WRONG: Editing gamemode/core/libs/sh_player.lua

-- RIGHT: Create a plugin with hooks
function PLUGIN:PlayerSpawn(client)
    -- Your custom logic here
end
```

**Don't forget realm checks:**
```lua
-- WRONG
function PLUGIN:PlayerInitialSpawn(client)
    client:ChatPrint("Welcome!")  -- Might run on client!
end

-- RIGHT
function PLUGIN:PlayerInitialSpawn(client)
    if SERVER then
        client:ChatPrint("Welcome!")
    end
end
```

**Don't trust client input:**
```lua
-- WRONG
net.Receive("GiveMoney", function(len, client)
    local amount = net.ReadInt(32)
    client:GetCharacter():GiveMoney(amount)  -- Client can send any amount!
end)

-- RIGHT
net.Receive("RequestMoney", function(len, client)
    -- Server validates and decides amount
    if client:IsAdmin() then
        local amount = 100  -- Server-controlled
        client:GetCharacter():GiveMoney(amount)
    end
end)
```

## Common Patterns

### Pattern 1: Storing Player-Specific Data

```lua
function PLUGIN:PlayerInitialSpawn(client)
    if SERVER then
        local steamID = client:SteamID()
        local data = self:GetData()

        -- Initialize player data if doesn't exist
        data.players = data.players or {}
        data.players[steamID] = data.players[steamID] or {
            joinCount = 0,
            lastJoin = os.time()
        }

        -- Update
        data.players[steamID].joinCount = data.players[steamID].joinCount + 1
        data.players[steamID].lastJoin = os.time()

        self:SetData(data)
    end
end
```

### Pattern 2: Configuration Options

```lua
-- In sh_plugin.lua or sh_config.lua
ix.config.Add("welcomeMessageEnabled", true, "Enable welcome messages", nil, {
    category = "Welcome Message"
})

ix.config.Add("welcomeMessageText", "Welcome!", "Message to show players", nil, {
    category = "Welcome Message"
})

-- Use in hooks
function PLUGIN:PlayerInitialSpawn(client)
    if SERVER and ix.config.Get("welcomeMessageEnabled") then
        local message = ix.config.Get("welcomeMessageText")
        timer.Simple(1, function()
            if IsValid(client) then
                client:ChatPrint(message:format(client:Name()))
            end
        end)
    end
end
```

### Pattern 3: Character-Specific Actions

```lua
function PLUGIN:PlayerLoadedCharacter(client, character, previousChar)
    if SERVER then
        -- This runs when player selects a character
        local charID = character:GetID()
        local data = character:GetData("welcomeShown")

        if not data then
            client:ChatPrint("Welcome, " .. character:GetName() .. "!")
            character:SetData("welcomeShown", true)
        end
    end
end
```

## Next Steps

Now that you've created your first plugin, explore:

1. **[Plugin Structure](plugin-structure.md)** - Learn about advanced directory structures
2. **[Available Hooks](hooks.md)** - Complete list of hooks you can use
3. **[Best Practices](best-practices.md)** - Guidelines for quality plugins
4. **[Plugin System Overview](plugin-system.md)** - Deep dive into how plugins work

## See Also

- [Plugin System](plugin-system.md) - Complete plugin system documentation
- [Command System](../systems/commands.md) - Creating chat commands
- [Configuration System](../systems/configuration.md) - Adding config options
- [Data System](../libraries/data.md) - Persistent data storage
- Source: `gamemode/core/libs/sh_plugin.lua`
