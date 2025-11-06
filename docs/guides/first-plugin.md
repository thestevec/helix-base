# Your First Plugin - Complete Tutorial

> **Reference**: `gamemode/core/libs/sh_plugin.lua`

Step-by-step guide to creating your first Helix plugin from scratch. You'll build a complete, working plugin that demonstrates core concepts.

## ⚠️ Important: Use Helix's Plugin System

**Always use Helix's built-in plugin system** rather than creating standalone Lua files. The framework provides:
- Automatic file loading and hot-reloading
- Hook system integration
- Data persistence per-plugin
- Configuration management
- Proper realm separation

## What You'll Build

A **Welcome Plugin** that:
- Welcomes players when they load a character
- Adds a `/playtime` command to show time played
- Tracks and displays player statistics
- Saves data persistently to database
- Demonstrates hooks, commands, and data storage

**Estimated time**: 15-20 minutes

## Prerequisites

- Helix framework installed
- Schema created (see [Setup Guide](setup.md))
- Basic Lua knowledge
- Text editor ready

## Step 1: Create Plugin Folder

Navigate to your schema's plugins directory and create a new folder:

```bash
cd garrysmod/gamemodes/myschema/plugins
mkdir welcome
cd welcome
```

**Directory structure:**
```
myschema/
└── plugins/
    └── welcome/          # Your new plugin
```

**⚠️ Important naming:**
- Use lowercase folder names
- No spaces (use underscores if needed)
- Folder name becomes the plugin's uniqueID

## Step 2: Create Main Plugin File

Create `sh_plugin.lua` (the "sh_" prefix means shared - runs on both server and client):

**plugins/welcome/sh_plugin.lua:**
```lua
-- Plugin metadata (required)
PLUGIN.name = "Welcome Plugin"
PLUGIN.author = "Your Name"
PLUGIN.description = "Welcomes players and tracks playtime"

-- This is a shared file that runs on both server and client
if SERVER then
    -- Server-only code will go here later
    print("[Welcome Plugin] Loaded on server")
end

if CLIENT then
    -- Client-only code will go here later
    print("[Welcome Plugin] Loaded on client")
end
```

## Step 3: Test Basic Plugin

**Reload your server:**
```
// In server console
sh_rebuildschema
```

**Check if it loaded:**
```lua
// In server console
lua_run PrintTable(ix.plugin.list)

// You should see "welcome" in the list
lua_run print(ix.plugin.list["welcome"].name)
// Output: "Welcome Plugin"
```

✅ **Checkpoint**: Your plugin is now recognized by Helix!

## Step 4: Add Welcome Message Hook

Hooks let your plugin respond to framework events. Let's add a welcome message when players load their character.

**Update plugins/welcome/sh_plugin.lua:**
```lua
PLUGIN.name = "Welcome Plugin"
PLUGIN.author = "Your Name"
PLUGIN.description = "Welcomes players and tracks playtime"

-- Hook: Called when player loads a character
function PLUGIN:PlayerLoadedCharacter(client, character, currentChar)
    -- This runs on server only (even though file is shared)
    if SERVER then
        -- Wait 1 second so player is fully loaded
        timer.Simple(1, function()
            -- Check if player is still valid
            if not IsValid(client) then
                return
            end

            -- Get character name
            local name = character:GetName()

            -- Send welcome message
            client:ChatPrint("Welcome back, " .. name .. "!")
            client:ChatPrint("Type /playtime to see your stats.")

            -- Play a sound (optional)
            client:EmitSound("buttons/button17.wav")
        end)
    end
end
```

**Reload and test:**
```
sh_rebuildschema
```

Then create or load a character. You should see the welcome message!

✅ **Checkpoint**: You've created your first hook!

## Step 5: Add Playtime Command

Let's add a command so players can check their playtime.

**Reference**: `gamemode/core/libs/sh_command.lua:38`

**Update plugins/welcome/sh_plugin.lua:**
```lua
PLUGIN.name = "Welcome Plugin"
PLUGIN.author = "Your Name"
PLUGIN.description = "Welcomes players and tracks playtime"

-- Register command after plugins are initialized
function PLUGIN:InitializedPlugins()
    -- Register the /playtime command
    ix.command.Add("PlayTime", {
        description = "Shows how long you've been playing",
        OnRun = function(self, client)
            -- Get the player's character
            local character = client:GetCharacter()

            if not character then
                return "You don't have a character loaded!"
            end

            -- Get saved playtime data (default to 0)
            local playTime = character:GetData("totalPlaytime", 0)

            -- Format time nicely
            local hours = math.floor(playTime / 3600)
            local minutes = math.floor((playTime % 3600) / 60)
            local seconds = playTime % 60

            -- Return message to player
            return string.format(
                "Playtime: %d hours, %d minutes, %d seconds",
                hours, minutes, seconds
            )
        end
    })
end

-- Welcome message (from before)
function PLUGIN:PlayerLoadedCharacter(client, character, currentChar)
    if SERVER then
        timer.Simple(1, function()
            if not IsValid(client) then return end

            local name = character:GetName()
            client:ChatPrint("Welcome back, " .. name .. "!")
            client:ChatPrint("Type /playtime to see your stats.")
            client:EmitSound("buttons/button17.wav")
        end)
    end
end
```

**Why use InitializedPlugins?**
- Commands must be registered AFTER the command system loads
- InitializedPlugins hook runs after all plugins are ready
- **Don't register commands at the top level of your file!**

**⚠️ Do NOT:**
```lua
-- WRONG - Runs too early!
ix.command.Add("PlayTime", {...})

PLUGIN.name = "Welcome"
```

**✅ DO:**
```lua
PLUGIN.name = "Welcome"

-- CORRECT - Runs after initialization
function PLUGIN:InitializedPlugins()
    ix.command.Add("PlayTime", {...})
end
```

**Test the command:**
```
sh_rebuildschema
```

Type `/playtime` in chat. You should see: `Playtime: 0 hours, 0 minutes, 0 seconds`

✅ **Checkpoint**: You've added a command!

## Step 6: Track Playtime

Now let's actually track how long players have been online.

**Update plugins/welcome/sh_plugin.lua:**
```lua
PLUGIN.name = "Welcome Plugin"
PLUGIN.author = "Your Name"
PLUGIN.description = "Welcomes players and tracks playtime"

-- Store when each character started playing
PLUGIN.sessionStartTimes = PLUGIN.sessionStartTimes or {}

-- Called when character loads
function PLUGIN:PlayerLoadedCharacter(client, character, currentChar)
    if SERVER then
        -- Track when this session started
        self.sessionStartTimes[character:GetID()] = os.time()

        timer.Simple(1, function()
            if not IsValid(client) then return end

            local name = character:GetName()
            client:ChatPrint("Welcome back, " .. name .. "!")
            client:ChatPrint("Type /playtime to see your stats.")
            client:EmitSound("buttons/button17.wav")
        end)
    end
end

-- Called before character is saved
function PLUGIN:CharacterPreSave(character)
    if SERVER then
        local charID = character:GetID()
        local startTime = self.sessionStartTimes[charID]

        if startTime then
            -- Calculate time played this session
            local sessionTime = os.time() - startTime

            -- Get previous total playtime
            local totalPlaytime = character:GetData("totalPlaytime", 0)

            -- Add this session's time
            character:SetData("totalPlaytime", totalPlaytime + sessionTime)

            -- Reset session start time
            self.sessionStartTimes[charID] = os.time()
        end
    end
end

-- Register command
function PLUGIN:InitializedPlugins()
    ix.command.Add("PlayTime", {
        description = "Shows how long you've been playing",
        OnRun = function(self, client)
            local character = client:GetCharacter()

            if not character then
                return "You don't have a character loaded!"
            end

            -- Get saved playtime
            local savedTime = character:GetData("totalPlaytime", 0)

            -- Get current session time
            local charID = character:GetID()
            local sessionStart = PLUGIN.sessionStartTimes[charID] or os.time()
            local sessionTime = os.time() - sessionStart

            -- Total time is saved + current session
            local playTime = savedTime + sessionTime

            local hours = math.floor(playTime / 3600)
            local minutes = math.floor((playTime % 3600) / 60)
            local seconds = playTime % 60

            return string.format(
                "Total Playtime: %d hours, %d minutes, %d seconds",
                hours, minutes, seconds
            )
        end
    })
end
```

**Test it:**
1. `sh_rebuildschema`
2. Play for a minute
3. Type `/playtime` - should show time played
4. Restart server
5. Type `/playtime` again - time should be saved!

✅ **Checkpoint**: You're tracking persistent data!

## Step 7: Add Configuration Options

Let's make the plugin configurable so server owners can customize it.

**Create plugins/welcome/sh_config.lua:**
```lua
-- Configuration for Welcome Plugin
-- Reference: gamemode/core/libs/sh_config.lua

-- Enable/disable welcome message
ix.config.Add("welcomeEnabled", true, "Enable welcome messages", function(oldValue, newValue)
    if SERVER then
        print("[Welcome] Welcome messages " .. (newValue and "enabled" or "disabled"))
    end
end, {
    category = "Welcome Plugin"
})

-- Custom welcome message format
ix.config.Add("welcomeMessage", "Welcome back, %s!", "Welcome message format (%s = name)", nil, {
    category = "Welcome Plugin"
})

-- Play sound on join
ix.config.Add("welcomeSound", "buttons/button17.wav", "Sound to play on welcome", nil, {
    category = "Welcome Plugin"
})

-- Playtime display format
ix.config.Add("playtimeFormat", "short", "Playtime format", nil, {
    category = "Welcome Plugin",
    type = "String",
    choices = {"short", "long"}
})
```

**Update plugins/welcome/sh_plugin.lua to use config:**
```lua
PLUGIN.name = "Welcome Plugin"
PLUGIN.author = "Your Name"
PLUGIN.description = "Welcomes players and tracks playtime"

PLUGIN.sessionStartTimes = PLUGIN.sessionStartTimes or {}

function PLUGIN:PlayerLoadedCharacter(client, character, currentChar)
    if SERVER then
        self.sessionStartTimes[character:GetID()] = os.time()

        -- Check if welcome messages are enabled
        if not ix.config.Get("welcomeEnabled", true) then
            return
        end

        timer.Simple(1, function()
            if not IsValid(client) then return end

            local name = character:GetName()

            -- Use custom message from config
            local messageFormat = ix.config.Get("welcomeMessage", "Welcome back, %s!")
            local message = string.format(messageFormat, name)

            client:ChatPrint(message)
            client:ChatPrint("Type /playtime to see your stats.")

            -- Use custom sound from config
            local sound = ix.config.Get("welcomeSound", "buttons/button17.wav")
            if sound and sound ~= "" then
                client:EmitSound(sound)
            end
        end)
    end
end

function PLUGIN:CharacterPreSave(character)
    if SERVER then
        local charID = character:GetID()
        local startTime = self.sessionStartTimes[charID]

        if startTime then
            local sessionTime = os.time() - startTime
            local totalPlaytime = character:GetData("totalPlaytime", 0)
            character:SetData("totalPlaytime", totalPlaytime + sessionTime)
            self.sessionStartTimes[charID] = os.time()
        end
    end
end

function PLUGIN:InitializedPlugins()
    ix.command.Add("PlayTime", {
        description = "Shows how long you've been playing",
        OnRun = function(self, client)
            local character = client:GetCharacter()
            if not character then
                return "You don't have a character loaded!"
            end

            local savedTime = character:GetData("totalPlaytime", 0)
            local charID = character:GetID()
            local sessionStart = PLUGIN.sessionStartTimes[charID] or os.time()
            local sessionTime = os.time() - sessionStart
            local playTime = savedTime + sessionTime

            -- Use format from config
            local format = ix.config.Get("playtimeFormat", "short")

            if format == "long" then
                -- Long format
                local hours = math.floor(playTime / 3600)
                local minutes = math.floor((playTime % 3600) / 60)
                local seconds = playTime % 60

                return string.format(
                    "You have played for %d hours, %d minutes, and %d seconds",
                    hours, minutes, seconds
                )
            else
                -- Short format
                local hours = math.floor(playTime / 3600)
                local minutes = math.floor((playTime % 3600) / 60)

                return string.format("Playtime: %dh %dm", hours, minutes)
            end
        end
    })
end
```

**Test configuration:**
```
sh_rebuildschema

// Change config via console
ix_config welcomeMessage "Hello again, %s!"
ix_config playtimeFormat "long"

// Or in server.cfg
ix_config welcomeMessage "Hello again, %s!"
```

✅ **Checkpoint**: Your plugin is now configurable!

## Step 8: Final Plugin Structure

Your complete plugin should now look like this:

```
plugins/welcome/
├── sh_plugin.lua      # Main plugin file
└── sh_config.lua      # Configuration options
```

## Complete Code

**plugins/welcome/sh_plugin.lua:**
```lua
PLUGIN.name = "Welcome Plugin"
PLUGIN.author = "Your Name"
PLUGIN.description = "Welcomes players and tracks playtime"

-- Session tracking table
PLUGIN.sessionStartTimes = PLUGIN.sessionStartTimes or {}

-- Called when character loads
function PLUGIN:PlayerLoadedCharacter(client, character, currentChar)
    if SERVER then
        -- Track session start time
        self.sessionStartTimes[character:GetID()] = os.time()

        -- Check if welcome messages enabled
        if not ix.config.Get("welcomeEnabled", true) then
            return
        end

        timer.Simple(1, function()
            if not IsValid(client) then return end

            local name = character:GetName()
            local messageFormat = ix.config.Get("welcomeMessage", "Welcome back, %s!")
            local message = string.format(messageFormat, name)

            client:ChatPrint(message)
            client:ChatPrint("Type /playtime to see your stats.")

            local sound = ix.config.Get("welcomeSound", "buttons/button17.wav")
            if sound and sound ~= "" then
                client:EmitSound(sound)
            end
        end)
    end
end

-- Save playtime before character saves
function PLUGIN:CharacterPreSave(character)
    if SERVER then
        local charID = character:GetID()
        local startTime = self.sessionStartTimes[charID]

        if startTime then
            local sessionTime = os.time() - startTime
            local totalPlaytime = character:GetData("totalPlaytime", 0)
            character:SetData("totalPlaytime", totalPlaytime + sessionTime)
            self.sessionStartTimes[charID] = os.time()
        end
    end
end

-- Register commands after initialization
function PLUGIN:InitializedPlugins()
    ix.command.Add("PlayTime", {
        description = "Shows how long you've been playing",
        OnRun = function(self, client)
            local character = client:GetCharacter()
            if not character then
                return "You don't have a character loaded!"
            end

            local savedTime = character:GetData("totalPlaytime", 0)
            local charID = character:GetID()
            local sessionStart = PLUGIN.sessionStartTimes[charID] or os.time()
            local sessionTime = os.time() - sessionStart
            local playTime = savedTime + sessionTime

            local format = ix.config.Get("playtimeFormat", "short")

            if format == "long" then
                local hours = math.floor(playTime / 3600)
                local minutes = math.floor((playTime % 3600) / 60)
                local seconds = playTime % 60

                return string.format(
                    "You have played for %d hours, %d minutes, and %d seconds",
                    hours, minutes, seconds
                )
            else
                local hours = math.floor(playTime / 3600)
                local minutes = math.floor((playTime % 3600) / 60)

                return string.format("Playtime: %dh %dm", hours, minutes)
            end
        end
    })
end
```

**plugins/welcome/sh_config.lua:**
```lua
ix.config.Add("welcomeEnabled", true, "Enable welcome messages", function(oldValue, newValue)
    if SERVER then
        print("[Welcome] Welcome messages " .. (newValue and "enabled" or "disabled"))
    end
end, {
    category = "Welcome Plugin"
})

ix.config.Add("welcomeMessage", "Welcome back, %s!", "Welcome message format (%s = name)", nil, {
    category = "Welcome Plugin"
})

ix.config.Add("welcomeSound", "buttons/button17.wav", "Sound to play on welcome", nil, {
    category = "Welcome Plugin"
})

ix.config.Add("playtimeFormat", "short", "Playtime format", nil, {
    category = "Welcome Plugin",
    type = "String",
    choices = {"short", "long"}
})
```

## Testing Checklist

Test your complete plugin:

- [ ] Plugin loads without errors after `sh_rebuildschema`
- [ ] Welcome message appears when loading character
- [ ] Sound plays on character load
- [ ] `/playtime` command works
- [ ] Playtime persists after server restart
- [ ] Config options can be changed
- [ ] Config changes affect plugin behavior

## Common Issues

### Plugin doesn't load

**Symptom**: No welcome message, command not found

**Causes**:
1. Wrong folder name or location
2. Syntax error in code
3. Missing `sh_plugin.lua` file

**Fix**:
```
// Check console for errors
con_filter_enable 1
con_filter_text "error"

// Verify plugin loaded
lua_run PrintTable(ix.plugin.list)
```

### Command not working

**Symptom**: "Unknown command" error

**Causes**:
1. Command registered too early (not in InitializedPlugins)
2. Typo in command name
3. Plugin didn't reload

**Fix**:
```lua
// ✅ CORRECT
function PLUGIN:InitializedPlugins()
    ix.command.Add("PlayTime", {...})
end

// ❌ WRONG
ix.command.Add("PlayTime", {...})  // Too early!
```

### Playtime not saving

**Symptom**: Playtime resets to 0 after restart

**Causes**:
1. CharacterPreSave hook not firing
2. Data not being set correctly
3. Character not saving

**Fix**:
```lua
-- Add debug prints
function PLUGIN:CharacterPreSave(character)
    print("[Welcome] Saving playtime for", character:GetName())
    -- rest of code
end
```

### Welcome message appears twice

**Symptom**: Multiple welcome messages

**Cause**: Hook running on both server and client

**Fix**:
```lua
// ✅ CORRECT
function PLUGIN:PlayerLoadedCharacter(client, character)
    if SERVER then  // Only run on server!
        -- send message
    end
end

// ❌ WRONG - runs on both realms
function PLUGIN:PlayerLoadedCharacter(client, character)
    -- send message
end
```

## Best Practices Learned

### ✅ DO

- Use `if SERVER` and `if CLIENT` for realm-specific code
- Register commands in `InitializedPlugins` hook
- Store data with `character:SetData()` for persistence
- Check if entities are valid before using them
- Use timers for delayed actions
- Add configuration options for flexibility
- Add descriptive comments

### ❌ DON'T

- Don't register commands at the top level of files
- Don't forget realm checks (SERVER/CLIENT)
- Don't access character data without validation
- Don't hardcode values - use config instead
- Don't skip error checking
- Don't forget to clean up timers/data

## Expanding Your Plugin

Now that you have a working plugin, try adding:

### More Commands
```lua
ix.command.Add("ResetPlaytime", {
    description = "Reset your playtime (admin only)",
    adminOnly = true,
    OnRun = function(self, client)
        local character = client:GetCharacter()
        character:SetData("totalPlaytime", 0)
        return "Playtime reset!"
    end
})
```

### More Hooks
```lua
function PLUGIN:PlayerDeath(victim, inflictor, attacker)
    local character = victim:GetCharacter()
    if character then
        local deaths = character:GetData("deaths", 0)
        character:SetData("deaths", deaths + 1)
    end
end
```

### Statistics Tracking
```lua
function PLUGIN:OnCharacterCreated(client, character)
    character:SetData("createdAt", os.time())
    character:SetData("deaths", 0)
    character:SetData("kills", 0)
end
```

## Next Steps

Congratulations! You've created a complete Helix plugin. Next, learn:

1. **[Custom Items Guide](custom-items.md)** - Create items for your plugin
2. **[Commands Guide](commands.md)** - Advanced command features
3. **[Hooks Guide](hooks.md)** - All available hooks
4. **[Database Guide](database.md)** - Working with databases
5. **[UI Development](ui-development.md)** - Create menus and panels

## See Also

- [Plugin System Documentation](../plugins/plugin-system.md)
- [Commands System](../systems/commands.md)
- [Character System](../systems/character.md)
- [Configuration System](../systems/configuration.md)
- Source: `gamemode/core/libs/sh_plugin.lua`
