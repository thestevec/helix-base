# Plugin Development Best Practices

> **Reference**: `gamemode/core/libs/sh_plugin.lua`

This guide provides best practices, patterns, and guidelines for creating high-quality Helix plugins.

## ⚠️ Important: Follow Framework Patterns

**Always use Helix's built-in systems** rather than creating custom implementations. The framework provides:
- Data persistence with `PLUGIN:SetData()` and `PLUGIN:GetData()`
- Configuration with `ix.config.Add()`
- Commands with `ix.command.Add()`
- Networking with `ix.net` functions
- Item, faction, class, and attribute registration

## Code Organization

### Use Clear Directory Structure

**✅ DO:**
```
plugins/myplugin/
├── sh_plugin.lua          # Metadata and initialization
├── sh_config.lua          # Configuration options
├── sh_commands.lua        # Commands
├── sv_hooks.lua           # Server hooks
├── cl_hooks.lua           # Client hooks
└── derma/                 # UI components
    └── cl_menu.lua
```

**❌ DON'T:**
```
plugins/myplugin/
└── everything.lua         # Everything in one file (hard to maintain)
```

### Separate Concerns

**✅ DO:**
```lua
-- sh_config.lua - Configuration
ix.config.Add("pluginEnabled", true, "Enable plugin", nil, {
    category = "My Plugin"
})

-- sh_commands.lua - Commands
ix.command.Add("MyCommand", {
    -- Command definition
})

-- sv_hooks.lua - Server logic
function PLUGIN:PlayerSpawn(client)
    -- Hook implementation
end
```

**❌ DON'T:**
```lua
-- sh_plugin.lua - Everything mixed together
PLUGIN.name = "My Plugin"

ix.config.Add(...)  -- Config
ix.command.Add(...) -- Command
function PLUGIN:PlayerSpawn(...) -- Hook
-- 1000 lines of mixed code
```

### Use Meaningful Names

**✅ DO:**
```lua
function PLUGIN:SavePlayerJoinData(client)
    local steamID = client:SteamID()
    local data = self:GetData()
    data.joinCounts[steamID] = (data.joinCounts[steamID] or 0) + 1
    self:SetData(data)
end

function PLUGIN:GetPlayerJoinCount(client)
    local data = self:GetData()
    return data.joinCounts[client:SteamID()] or 0
end
```

**❌ DON'T:**
```lua
function PLUGIN:DoStuff(c)
    local d = self:GetData()
    d.x[c:SteamID()] = (d.x[c:SteamID()] or 0) + 1
    self:SetData(d)
end

function PLUGIN:GetX(c)
    return self:GetData().x[c:SteamID()] or 0
end
```

## Realm Management

### Always Check Realm

**✅ DO:**
```lua
function PLUGIN:PlayerInitialSpawn(client)
    if SERVER then
        -- Server-only code
        client:ChatPrint("Welcome!")
    end

    if CLIENT then
        -- Client-only code
        chat.AddText("Player spawned")
    end
end
```

**❌ DON'T:**
```lua
function PLUGIN:PlayerInitialSpawn(client)
    client:ChatPrint("Welcome!")  -- Might run on client and error!
    chat.AddText("Player spawned") -- Might run on server and error!
end
```

### Use Proper File Prefixes

**✅ DO:**
```
sv_database.lua    # Server: database operations
cl_menu.lua        # Client: UI code
sh_config.lua      # Shared: config definitions
```

**❌ DON'T:**
```
database.lua       # Ambiguous realm
menu.lua           # Ambiguous realm
config.lua         # Ambiguous realm
```

### Network Properly

**✅ DO:**
```lua
-- Server: sv_networking.lua
util.AddNetworkString("MyPlugin.UpdateData")

function PLUGIN:SendDataToClient(client, data)
    net.Start("MyPlugin.UpdateData")
        net.WriteTable(data)
    net.Send(client)
end

-- Client: cl_networking.lua
net.Receive("MyPlugin.UpdateData", function()
    local data = net.ReadTable()
    PLUGIN:HandleData(data)
end)
```

**❌ DON'T:**
```lua
-- Mixing server and client in shared file without checks
function PLUGIN:SendData(client, data)
    net.Start("UpdateData") -- No namespace!
    net.WriteTable(data)
    net.Send(client)
end
```

## Performance Optimization

### Cache Expensive Operations

**✅ DO:**
```lua
function PLUGIN:InitializedPlugins()
    -- Cache config values
    self.enabled = ix.config.Get("myPluginEnabled")
    self.interval = ix.config.Get("myPluginInterval")

    -- Cache expensive lookups
    self.adminSteamIDs = {}
    for _, v in ipairs(player.GetAll()) do
        if v:IsAdmin() then
            self.adminSteamIDs[v:SteamID()] = true
        end
    end
end

function PLUGIN:Think()
    if not self.enabled then return end

    if CurTime() > (self.nextThink or 0) then
        self.nextThink = CurTime() + self.interval
        self:DoPeriodicTask()
    end
end
```

**❌ DON'T:**
```lua
function PLUGIN:Think()
    -- Expensive config lookup every tick!
    if not ix.config.Get("myPluginEnabled") then return end

    -- Iterating all players every tick!
    for _, v in ipairs(player.GetAll()) do
        if v:IsAdmin() then
            -- Do something
        end
    end
end
```

### Use Think Hook Sparingly

**✅ DO:**
```lua
function PLUGIN:InitializedPlugins()
    if SERVER then
        -- Use timer for periodic tasks
        timer.Create("MyPlugin.PeriodicTask", 60, 0, function()
            self:DoPeriodicTask()
        end)
    end
end

function PLUGIN:OnPluginUnloaded()
    timer.Remove("MyPlugin.PeriodicTask")
end
```

**❌ DON'T:**
```lua
function PLUGIN:Think()
    -- Running every tick (66+ times per second!)
    self:DoExpensiveOperation()
end
```

### Avoid Unnecessary Loops

**✅ DO:**
```lua
function PLUGIN:FindPlayerByName(name)
    for _, client in ipairs(player.GetAll()) do
        if client:Name() == name then
            return client -- Stop when found
        end
    end
end

-- Or use Helix helper
local client = ix.util.FindPlayer(name)
```

**❌ DON'T:**
```lua
function PLUGIN:FindPlayerByName(name)
    local found = nil
    for _, client in ipairs(player.GetAll()) do
        if client:Name() == name then
            found = client
            -- Keeps looping even after found!
        end
    end
    return found
end
```

### Optimize Network Traffic

**✅ DO:**
```lua
-- Send only necessary data
function PLUGIN:SendInventoryUpdate(client, itemID, amount)
    net.Start("MyPlugin.ItemUpdate")
        net.WriteUInt(itemID, 16)
        net.WriteUInt(amount, 8)
    net.Send(client)
end

-- Batch updates
function PLUGIN:SendMultipleUpdates(client, updates)
    net.Start("MyPlugin.BatchUpdate")
        net.WriteUInt(#updates, 8)
        for _, update in ipairs(updates) do
            net.WriteUInt(update.id, 16)
            net.WriteUInt(update.amount, 8)
        end
    net.Send(client)
end
```

**❌ DON'T:**
```lua
-- Sending entire table every update
function PLUGIN:SendInventoryUpdate(client)
    net.Start("MyPlugin.Update")
        net.WriteTable(self.allInventoryData) -- Huge!
    net.Send(client)
end

-- Sending same data repeatedly
function PLUGIN:Think()
    for _, client in ipairs(player.GetAll()) do
        self:SendInventoryUpdate(client) -- Every tick!
    end
end
```

## Security Best Practices

### Never Trust Client Input

**✅ DO:**
```lua
-- Server
util.AddNetworkString("MyPlugin.BuyItem")

net.Receive("MyPlugin.BuyItem", function(len, client)
    local itemID = net.ReadString()

    -- SERVER validates everything
    local item = ix.item.list[itemID]
    if not item then return end

    local character = client:GetCharacter()
    if not character then return end

    -- Check if player can afford
    if not character:HasMoney(item.price) then
        client:Notify("You can't afford this!")
        return
    end

    -- Server decides final action
    character:TakeMoney(item.price)
    character:GetInventory():Add(itemID)
end)
```

**❌ DON'T:**
```lua
-- Client decides price and gives item!
net.Receive("MyPlugin.BuyItem", function(len, client)
    local itemID = net.ReadString()
    local price = net.ReadInt(32) -- CLIENT SENT THIS!

    local character = client:GetCharacter()
    character:TakeMoney(price) -- Client could send 0!
    character:GetInventory():Add(itemID)
end)
```

### Validate All Data

**✅ DO:**
```lua
function PLUGIN:SetPlayerData(client, key, value)
    -- Whitelist allowed keys
    local allowedKeys = {
        ["preference"] = true,
        ["setting"] = true
    }

    if not allowedKeys[key] then
        ErrorNoHalt("Invalid key: " .. tostring(key))
        return
    end

    -- Validate value type
    if not isstring(value) or #value > 100 then
        ErrorNoHalt("Invalid value")
        return
    end

    client:GetCharacter():SetData(key, value)
end
```

**❌ DON'T:**
```lua
function PLUGIN:SetPlayerData(client, key, value)
    -- No validation!
    client:GetCharacter():SetData(key, value)
end
```

### Protect Admin Functions

**✅ DO:**
```lua
ix.command.Add("GiveMoney", {
    description = "Give money to a player",
    adminOnly = true, -- Built-in admin check
    arguments = {
        ix.type.player,
        ix.type.number
    },
    OnRun = function(self, client, target, amount)
        -- Additional validation
        if amount < 0 or amount > 1000000 then
            return "Invalid amount"
        end

        target:GetCharacter():GiveMoney(amount)
        return "Gave $" .. amount
    end
})
```

**❌ DON'T:**
```lua
ix.command.Add("GiveMoney", {
    description = "Give money to a player",
    -- No adminOnly check!
    OnRun = function(self, client, target, amount)
        target:GetCharacter():GiveMoney(amount) -- Anyone can use!
    end
})
```

### Sanitize SQL Queries

**✅ DO:**
```lua
function PLUGIN:GetPlayerStats(steamID)
    -- Use prepared statements
    local query = ix.db.Query("SELECT * FROM plugin_stats WHERE steamid = ?")
    query:SetParameter(1, steamID) -- Automatically escaped
    query:Execute()
end

-- Or use Helix character data (recommended)
function PLUGIN:SavePlayerStats(client, stats)
    local character = client:GetCharacter()
    character:SetData("stats", stats) -- Framework handles storage
end
```

**❌ DON'T:**
```lua
function PLUGIN:GetPlayerStats(steamID)
    -- SQL injection vulnerability!
    local query = "SELECT * FROM plugin_stats WHERE steamid = '" .. steamID .. "'"
    ix.db.Query(query):Execute()
end
```

## Data Management

### Use Framework Data Storage

**✅ DO:**
```lua
-- Plugin-level data
function PLUGIN:InitializedPlugins()
    if SERVER then
        local data = self:GetData()
        self.joinCounts = data.joinCounts or {}
    end
end

function PLUGIN:SaveJoinCount(steamID, count)
    self.joinCounts[steamID] = count

    local data = self:GetData()
    data.joinCounts = self.joinCounts
    self:SetData(data)
end

-- Character-level data
function PLUGIN:SaveCharacterPreference(character, preference)
    character:SetData("pluginPreference", preference)
end
```

**❌ DON'T:**
```lua
-- Manual file I/O (error-prone)
function PLUGIN:SaveData()
    local data = util.TableToJSON(self.data)
    file.Write("myplugin_data.txt", data)
end

-- Global variables (not saved)
MyPluginData = {}
```

### Initialize Data Structures

**✅ DO:**
```lua
function PLUGIN:InitializedPlugins()
    if SERVER then
        local data = self:GetData()

        -- Ensure structure exists
        data.players = data.players or {}
        data.stats = data.stats or {}
        data.version = data.version or 1

        self:SetData(data)
    end
end
```

**❌ DON'T:**
```lua
function PLUGIN:SomeHook(client)
    local data = self:GetData()
    -- Might be nil!
    data.players[client:SteamID()].score = 100
end
```

### Clean Up Old Data

**✅ DO:**
```lua
function PLUGIN:LoadData()
    local data = self:GetData()

    -- Remove data for players not seen in 30 days
    local cutoff = os.time() - (30 * 24 * 60 * 60)

    for steamID, playerData in pairs(data.players or {}) do
        if (playerData.lastSeen or 0) < cutoff then
            data.players[steamID] = nil
        end
    end

    self:SetData(data)
end
```

**❌ DON'T:**
```lua
function PLUGIN:PlayerInitialSpawn(client)
    local data = self:GetData()
    -- Endlessly accumulating data!
    data.players = data.players or {}
    data.players[client:SteamID()] = {}
    self:SetData(data)
end
```

## Error Handling

### Use Protected Calls for Risky Operations

**✅ DO:**
```lua
function PLUGIN:DoRiskyOperation(client)
    local success, result = pcall(function()
        -- Code that might error
        local data = util.JSONToTable(someString)
        return self:ProcessData(data)
    end)

    if not success then
        ErrorNoHalt("[" .. self.name .. "] Error: " .. tostring(result) .. "\n")
        return nil
    end

    return result
end
```

**❌ DON'T:**
```lua
function PLUGIN:DoRiskyOperation(client)
    -- Uncaught error will crash the hook chain!
    local data = util.JSONToTable(someString)
    return self:ProcessData(data)
end
```

### Validate Entity Existence

**✅ DO:**
```lua
function PLUGIN:DoSomethingWithPlayer(client)
    if not IsValid(client) then return end
    if not client:Alive() then return end

    local character = client:GetCharacter()
    if not character then return end

    -- Safe to proceed
    character:GiveMoney(100)
end

function PLUGIN:DelayedAction(entity)
    timer.Simple(5, function()
        if not IsValid(entity) then return end
        -- Entity still exists
        entity:Remove()
    end)
end
```

**❌ DON'T:**
```lua
function PLUGIN:DoSomethingWithPlayer(client)
    -- No validation!
    client:GetCharacter():GiveMoney(100) -- Crash if no character!
end

function PLUGIN:DelayedAction(entity)
    timer.Simple(5, function()
        entity:Remove() -- Entity might be gone!
    end)
end
```

### Provide Helpful Error Messages

**✅ DO:**
```lua
function PLUGIN:AddCustomItem(uniqueID, data)
    if not uniqueID then
        ErrorNoHalt("[" .. self.name .. "] AddCustomItem: uniqueID is required\n")
        return false
    end

    if not istable(data) then
        ErrorNoHalt("[" .. self.name .. "] AddCustomItem: data must be a table, got " .. type(data) .. "\n")
        return false
    end

    if not data.name then
        ErrorNoHalt("[" .. self.name .. "] AddCustomItem: data.name is required for " .. uniqueID .. "\n")
        return false
    end

    -- Proceed
    return true
end
```

**❌ DON'T:**
```lua
function PLUGIN:AddCustomItem(uniqueID, data)
    -- Silent failure or cryptic error
    self.items[uniqueID] = data
end
```

## Resource Cleanup

### Clean Up Timers

**✅ DO:**
```lua
function PLUGIN:InitializedPlugins()
    if SERVER then
        -- Use unique timer names with plugin identifier
        timer.Create("MyPlugin.PeriodicCheck", 60, 0, function()
            self:DoPeriodicCheck()
        end)
    end
end

function PLUGIN:OnPluginUnloaded()
    if SERVER then
        timer.Remove("MyPlugin.PeriodicCheck")
    end
end
```

**❌ DON'T:**
```lua
function PLUGIN:InitializedPlugins()
    timer.Create("Check", 60, 0, function() -- Generic name!
        self:DoPeriodicCheck()
    end)
end

-- No cleanup! Timer keeps running after plugin unloads
```

### Clean Up UI Elements

**✅ DO:**
```lua
function PLUGIN:CreateMenu()
    if IsValid(self.menuPanel) then
        self.menuPanel:Remove()
    end

    self.menuPanel = vgui.Create("DFrame")
    -- Setup menu
end

function PLUGIN:OnPluginUnloaded()
    if CLIENT and IsValid(self.menuPanel) then
        self.menuPanel:Remove()
        self.menuPanel = nil
    end
end
```

**❌ DON'T:**
```lua
function PLUGIN:CreateMenu()
    -- Creates new panel every time, old ones leak!
    self.menuPanel = vgui.Create("DFrame")
end

-- No cleanup
```

### Remove Hooks and Network Receivers

**✅ DO:**
```lua
-- Plugin hooks are automatically cleaned up by framework

-- For manual hooks:
function PLUGIN:InitializedPlugins()
    if CLIENT then
        hook.Add("PlayerButtonDown", "MyPlugin.ButtonHandler", function(ply, button)
            self:HandleButton(button)
        end)
    end
end

function PLUGIN:OnPluginUnloaded()
    if CLIENT then
        hook.Remove("PlayerButtonDown", "MyPlugin.ButtonHandler")
    end
end
```

## Configuration Management

### Provide Sensible Defaults

**✅ DO:**
```lua
ix.config.Add("myPluginEnabled", true, "Enable the plugin", nil, {
    category = "My Plugin"
})

ix.config.Add("myPluginInterval", 60, "Update interval in seconds", nil, {
    data = {min = 10, max = 600},
    category = "My Plugin"
})

ix.config.Add("myPluginMessage", "Welcome!", "Message to display", nil, {
    category = "My Plugin"
})
```

**❌ DON'T:**
```lua
-- No defaults, confusing names
ix.config.Add("mpe", nil)
ix.config.Add("mpi", nil)
ix.config.Add("mpm", nil)
```

### Use Config Callbacks

**✅ DO:**
```lua
ix.config.Add("myPluginInterval", 60, "Update interval", function(oldValue, newValue)
    -- Update timer when config changes
    if SERVER and timer.Exists("MyPlugin.Update") then
        timer.Adjust("MyPlugin.Update", newValue)
    end
end, {
    data = {min = 10, max = 600},
    category = "My Plugin"
})
```

### Group Related Configs

**✅ DO:**
```lua
-- All configs in sh_config.lua with same category
ix.config.Add("roleplay_enabled", true, "...", nil, {category = "Roleplay"})
ix.config.Add("roleplay_distance", 300, "...", nil, {category = "Roleplay"})
ix.config.Add("roleplay_format", "%s says \"%s\"", "...", nil, {category = "Roleplay"})
```

## Documentation

### Document Your Code

**✅ DO:**
```lua
--- Gives a player bonus money based on their playtime.
-- This function calculates bonus money using the formula: base * (hours / 10)
-- @param client Player The player to give money to
-- @param hours number How many hours the player has been online
-- @return number The amount of money given
function PLUGIN:GiveBonusMoney(client, hours)
    local base = ix.config.Get("bonusMoneyBase", 100)
    local bonus = base * (hours / 10)

    local character = client:GetCharacter()
    if character then
        character:GiveMoney(bonus)
    end

    return bonus
end
```

**❌ DON'T:**
```lua
function PLUGIN:GBM(c, h)
    local b = ix.config.Get("bmb", 100)
    local bn = b * (h / 10)
    c:GetCharacter():GiveMoney(bn)
    return bn
end
```

### Add README to Your Plugin

**✅ DO:**
```
plugins/myplugin/
├── README.md              # Documentation!
├── sh_plugin.lua
└── ...
```

**README.md:**
```markdown
# My Plugin

Description of what your plugin does.

## Features
- Feature 1
- Feature 2

## Configuration
- `myPluginEnabled` - Enable/disable plugin
- `myPluginInterval` - Update interval

## Commands
- `/mycommand <target>` - Does something

## Installation
1. Place in `schema/plugins/myplugin`
2. Restart server
3. Configure in F1 menu

## Dependencies
- None (or list required plugins)
```

## Testing

### Test All Realms

**✅ DO:**
```lua
-- Test server
function PLUGIN:PlayerSpawn(client)
    if SERVER then
        print("[SERVER] PlayerSpawn:", client:Name())
    end
end

-- Test client
function PLUGIN:PlayerSpawn(client)
    if CLIENT then
        print("[CLIENT] PlayerSpawn:", client:Name())
    end
end

-- Test both
function PLUGIN:InitializedPlugins()
    print("[" .. (SERVER and "SERVER" or "CLIENT") .. "] Plugin loaded")
end
```

### Test Edge Cases

**✅ DO:**
```lua
function PLUGIN:GiveItem(client, itemID, amount)
    -- Test: No client
    if not IsValid(client) then return false end

    -- Test: No character
    local character = client:GetCharacter()
    if not character then return false end

    -- Test: Invalid item
    local item = ix.item.list[itemID]
    if not item then return false end

    -- Test: Invalid amount
    amount = math.Clamp(amount or 1, 1, 100)

    -- Test: No inventory space
    local inventory = character:GetInventory()
    if not inventory:CanItemFit(item) then
        client:Notify("Inventory full!")
        return false
    end

    -- Proceed
    inventory:Add(itemID, amount)
    return true
end
```

## Common Mistakes to Avoid

### Mistake 1: Global Variables

**❌ WRONG:**
```lua
myPluginData = {}  -- Global!
```

**✅ RIGHT:**
```lua
PLUGIN.data = {}  -- Scoped to plugin
-- Or use self:SetData() for persistence
```

### Mistake 2: Not Checking Realms

**❌ WRONG:**
```lua
function PLUGIN:DoSomething()
    client:ChatPrint("Hello") -- SERVER function
    chat.AddText("Hello") -- CLIENT function
end
```

**✅ RIGHT:**
```lua
function PLUGIN:DoSomething()
    if SERVER then
        client:ChatPrint("Hello")
    end

    if CLIENT then
        chat.AddText("Hello")
    end
end
```

### Mistake 3: Memory Leaks

**❌ WRONG:**
```lua
function PLUGIN:CreatePanel()
    self.panel = vgui.Create("DFrame")
    -- Old panel never removed!
end
```

**✅ RIGHT:**
```lua
function PLUGIN:CreatePanel()
    if IsValid(self.panel) then
        self.panel:Remove()
    end

    self.panel = vgui.Create("DFrame")
end
```

### Mistake 4: Trusting Client

**❌ WRONG:**
```lua
net.Receive("BuyItem", function(len, client)
    local price = net.ReadInt(32) -- Client sent this!
    client:GetCharacter():TakeMoney(price)
end)
```

**✅ RIGHT:**
```lua
net.Receive("BuyItem", function(len, client)
    local itemID = net.ReadString()

    -- Server validates and decides price
    local item = ix.item.list[itemID]
    if item and client:GetCharacter():HasMoney(item.price) then
        client:GetCharacter():TakeMoney(item.price)
    end
end)
```

### Mistake 5: No Error Handling

**❌ WRONG:**
```lua
function PLUGIN:LoadConfig()
    local data = util.JSONToTable(file.Read("config.txt"))
    self.config = data.settings.advanced.values
end
```

**✅ RIGHT:**
```lua
function PLUGIN:LoadConfig()
    if not file.Exists("config.txt", "DATA") then
        self:CreateDefaultConfig()
        return
    end

    local contents = file.Read("config.txt", "DATA")
    if not contents then return end

    local success, data = pcall(util.JSONToTable, contents)
    if not success or not data then
        ErrorNoHalt("[" .. self.name .. "] Failed to load config\n")
        return
    end

    self.config = data.settings and data.settings.advanced and data.settings.advanced.values or {}
end
```

## See Also

- [Creating Plugins](creating-plugins.md) - Tutorial
- [Plugin System](plugin-system.md) - System overview
- [Plugin Structure](plugin-structure.md) - Directory organization
- [Hooks](hooks.md) - Available hooks
- [Command System](../systems/commands.md) - Creating commands
- [Configuration](../systems/configuration.md) - Config options
- Source: `gamemode/core/libs/sh_plugin.lua`
