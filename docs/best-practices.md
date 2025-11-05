# Helix Framework Best Practices

This guide covers best practices, coding standards, and recommendations for developing with the Helix framework.

## Table of Contents

- [Code Organization](#code-organization)
- [Naming Conventions](#naming-conventions)
- [Realm Separation](#realm-separation)
- [Performance](#performance)
- [Database Operations](#database-operations)
- [Networking](#networking)
- [Hook Usage](#hook-usage)
- [UI Development](#ui-development)
- [Security](#security)
- [Plugin Development](#plugin-development)
- [Error Handling](#error-handling)
- [Documentation](#documentation)

## Code Organization

### File Structure

**Follow the naming convention for realm prefixes:**

```
sh_  - Shared (runs on both client and server)
cl_  - Client only
sv_  - Server only
```

**Good:**
```
plugins/myplugin/
├── sh_plugin.lua      # Main plugin file (shared)
├── sv_hooks.lua       # Server-side hooks
├── cl_hooks.lua       # Client-side hooks
├── sh_commands.lua    # Commands (shared)
└── derma/
    └── cl_menu.lua    # UI (client only)
```

**Bad:**
```
plugins/myplugin/
├── plugin.lua         # Unclear realm
├── hooks.lua          # Unclear realm
└── menu.lua           # Unclear realm
```

### Include Order

Always include files in this order:

1. Libraries
2. Shared code
3. Server code (server-side)
4. Client code (AddCSLuaFile then include on client)

```lua
-- In plugin sh_plugin.lua
ix.util.Include("libs/sh_mylib.lua")        # Libraries first
ix.util.Include("sh_config.lua")            # Shared config
ix.util.Include("sv_hooks.lua")             # Server code
ix.util.Include("cl_hooks.lua")             # Client code
```

### Directory Organization

Use subdirectories to organize features:

```
plugins/myplugin/
├── sh_plugin.lua
├── items/              # Item definitions
│   ├── sh_weapon.lua
│   └── sh_armor.lua
├── factions/           # Faction definitions
│   └── sh_rebels.lua
├── classes/            # Class definitions
│   └── sh_medic.lua
├── derma/              # UI components
│   ├── cl_menu.lua
│   └── cl_panel.lua
├── entities/           # Custom entities
│   └── entities/
│       └── ent_custom.lua
└── languages/          # Localizations
    ├── sh_english.lua
    └── sh_spanish.lua
```

## Naming Conventions

### Variables

```lua
-- Local variables: camelCase
local myVariable = 5
local playerHealth = 100

-- Global variables: UPPER_SNAKE_CASE (avoid when possible)
MY_GLOBAL_VAR = "value"

-- Table keys: lowercase or camelCase
local data = {
    name = "John",
    playerCount = 5,
    isActive = true
}
```

### Functions

```lua
-- Local functions: camelCase
local function calculateDamage(amount)
    return amount * 2
end

-- Library functions: CamelCase or camelCase
function ix.util.GetPlayerName(player)
    return player:Name()
end

-- Meta methods: CamelCase
function PLAYER:GetCharacterName()
    local char = self:GetCharacter()
    return char and char:GetName() or self:Name()
end
```

### Files

```lua
-- Plugin files
sh_plugin.lua          # Main plugin
sh_commands.lua        # Commands
sh_config.lua          # Configuration
sv_database.lua        # Database operations
cl_menu.lua            # UI menu

-- Item files
sh_weapon_pistol.lua   # Specific item
sh_armor_vest.lua      # Specific item

-- Entity files
ent_myentity.lua       # Custom entity
```

### Plugin Structure

```lua
PLUGIN.name = "My Plugin"              -- Human-readable name
PLUGIN.author = "Your Name"             -- Your name
PLUGIN.description = "Description"      -- What it does
PLUGIN.version = "1.0.0"               -- Semantic versioning

-- Unique ID is folder name, use lowercase with no spaces
-- Good: "myplugin", "my_plugin", "advancedvendors"
-- Bad: "My Plugin", "myPlugin", "My-Plugin"
```

## Realm Separation

### Always Check Realm

```lua
-- Good: Check realm before running code
if SERVER then
    function PLUGIN:PlayerSpawn(client)
        -- Server-only code
    end
end

if CLIENT then
    function PLUGIN:HUDPaint()
        -- Client-only code
    end
end

-- For shared code, no check needed
function PLUGIN:PlayerSay(client, text)
    -- Runs on both client and server
    return text
end
```

### Don't Mix Realms

```lua
-- Bad: Client code on server
function PLUGIN:PlayerSpawn(client)
    client:ConCommand("say Hello")  -- ConCommand is client-only!
end

-- Good: Use proper functions
function PLUGIN:PlayerSpawn(client)
    if SERVER then
        client:ChatPrint("Welcome!")  -- Works on server
    end
end
```

### Network for Cross-Realm Communication

```lua
-- Server to Client
if SERVER then
    util.AddNetworkString("MyMessage")

    function PLUGIN:SendToClient(client, data)
        net.Start("MyMessage")
        net.WriteString(data)
        net.Send(client)
    end
end

-- Client receives
if CLIENT then
    net.Receive("MyMessage", function()
        local data = net.ReadString()
        print("Received:", data)
    end)
end
```

## Performance

### Cache Frequently Used Values

```lua
-- Bad: Repeated function calls
for i = 1, 100 do
    if CurTime() > nextUpdate then
        -- CurTime() called 100 times
    end
end

-- Good: Cache the value
local curTime = CurTime()
for i = 1, 100 do
    if curTime > nextUpdate then
        -- CurTime() called once
    end
end
```

### Avoid Repeated Lookups

```lua
-- Bad: Table lookup in loop
for _, player in ipairs(player.GetAll()) do
    player:SetNWString("faction", ix.faction.indices[player:GetCharacter():GetFaction()].name)
end

-- Good: Cache lookups
local factionIndices = ix.faction.indices
for _, player in ipairs(player.GetAll()) do
    local char = player:GetCharacter()
    if char then
        local faction = factionIndices[char:GetFaction()]
        if faction then
            player:SetNWString("faction", faction.name)
        end
    end
end
```

### Limit Hook Frequency

```lua
-- Bad: Heavy processing every tick
function PLUGIN:Think()
    -- Runs every tick (66+ times per second)
    for _, player in ipairs(player.GetAll()) do
        -- Expensive operation
    end
end

-- Good: Throttle updates
PLUGIN.nextThink = 0

function PLUGIN:Think()
    local curTime = CurTime()
    if curTime < self.nextThink then return end
    self.nextThink = curTime + 1  -- Run once per second

    for _, player in ipairs(player.GetAll()) do
        -- Expensive operation
    end
end
```

### Use Local Variables

```lua
-- Bad: Global lookup every time
function myFunction()
    print(ix.config.Get("serverName"))  -- Global lookup
end

-- Good: Cache as local
local function myFunction()
    local serverName = ix.config.Get("serverName")
    print(serverName)
end
```

## Database Operations

### Always Use Prepared Statements

```lua
-- Bad: SQL injection vulnerable
local steamID = player:SteamID()
ix.db.Query("SELECT * FROM ix_players WHERE _steamID = '" .. steamID .. "'")

-- Good: Prepared statement
ix.db.Query("SELECT * FROM ix_players WHERE _steamID = ?", function(data)
    -- Process data
end, steamID)
```

### Batch Database Operations

```lua
-- Bad: Multiple queries
for _, item in ipairs(items) do
    ix.db.Query("INSERT INTO ix_items VALUES (?, ?)", nil, item.id, item.name)
end

-- Good: Single batch query
local values = {}
for _, item in ipairs(items) do
    table.insert(values, string.format("(%d, '%s')", item.id, item.name))
end
ix.db.Query("INSERT INTO ix_items VALUES " .. table.concat(values, ", "))
```

### Use Transactions for Multiple Queries

```lua
-- Good: Use transactions for data consistency
ix.db.Query("BEGIN TRANSACTION")
ix.db.Query("UPDATE ix_characters SET _money = _money - ? WHERE _id = ?", nil, amount, buyerID)
ix.db.Query("UPDATE ix_characters SET _money = _money + ? WHERE _id = ?", nil, amount, sellerID)
ix.db.Query("COMMIT")
```

### Handle Query Failures

```lua
-- Bad: No error handling
ix.db.Query("SELECT * FROM ix_players", function(data)
    local player = data[1]  -- Could be nil!
    print(player.name)
end)

-- Good: Check for data
ix.db.Query("SELECT * FROM ix_players WHERE _steamID = ?", function(data)
    if not data or #data == 0 then
        print("No player found")
        return
    end

    local player = data[1]
    print(player._steamName)
end, steamID)
```

## Networking

### Minimize Network Messages

```lua
-- Bad: Send many small messages
for i = 1, 10 do
    net.Start("UpdateValue")
    net.WriteInt(i, 32)
    net.Send(client)
end

-- Good: Send one message with all data
net.Start("UpdateValues")
net.WriteTable({1, 2, 3, 4, 5, 6, 7, 8, 9, 10})
net.Send(client)
```

### Use Appropriate Data Types

```lua
-- Bad: Waste bandwidth
net.WriteString(tostring(123))      -- "123" = 4 bytes
net.WriteString(tostring(true))     -- "true" = 5 bytes

-- Good: Use correct types
net.WriteInt(123, 32)               -- 4 bytes
net.WriteBool(true)                 -- 1 bit
```

### Validate Client Data

```lua
-- Bad: Trust client
net.Receive("BuyItem", function(len, client)
    local itemID = net.ReadString()
    local price = net.ReadInt(32)

    -- Client could send negative price!
    client:GetCharacter():TakeMoney(price)
end)

-- Good: Validate on server
net.Receive("BuyItem", function(len, client)
    local itemID = net.ReadString()
    local clientPrice = net.ReadInt(32)

    -- Get real price from server
    local item = ix.item.list[itemID]
    if not item then return end

    local realPrice = item.price or 0
    if realPrice < 0 then return end

    local char = client:GetCharacter()
    if not char or not char:HasMoney(realPrice) then return end

    char:TakeMoney(realPrice)
    char:GetInventory():Add(itemID)
end)
```

## Hook Usage

### Use Appropriate Hooks

```lua
-- Bad: Wrong hook for the job
function PLUGIN:Think()
    -- Checking every tick is expensive
    for _, ply in ipairs(player.GetAll()) do
        if not ply:Alive() then
            ply:Spawn()
        end
    end
end

-- Good: Use specific hook
function PLUGIN:PlayerDeath(victim, inflictor, attacker)
    timer.Simple(5, function()
        if IsValid(victim) then
            victim:Spawn()
        end
    end)
end
```

### Don't Block Return Values

```lua
-- Bad: Blocks other hooks
function PLUGIN:PlayerSay(client, text)
    if text == "!help" then
        -- Do something
        return ""  -- Blocks text from being sent
    end
    -- Forgot to return text for other messages!
end

-- Good: Only return when needed
function PLUGIN:PlayerSay(client, text)
    if text == "!help" then
        -- Show help
        return ""  -- Block this specific message
    end
    -- Implicitly returns nil, allowing text through
end
```

### Order Matters

```lua
-- Hooks are called in order they were added
-- Return values stop the chain

function PLUGIN:CanPlayerUseItem(client, item)
    if item.uniqueID == "special_item" then
        return false  -- Blocks this item
        -- No other hooks are called for this item
    end
    -- Return nil to let other hooks decide
end
```

## UI Development

### Cache Colors and Fonts

```lua
-- Bad: Create color every paint
function PANEL:Paint(w, h)
    draw.RoundedBox(4, 0, 0, w, h, Color(50, 50, 50))
end

-- Good: Cache color
local backgroundColor = Color(50, 50, 50)
function PANEL:Paint(w, h)
    draw.RoundedBox(4, 0, 0, w, h, backgroundColor)
end
```

### Don't Call Layout Functions in Paint

```lua
-- Bad: Causes infinite loop
function PANEL:Paint(w, h)
    self:SetSize(w + 1, h)  -- SetSize calls Paint again!
end

-- Good: Use PerformLayout
function PANEL:PerformLayout(w, h)
    -- Layout code here
end

function PANEL:Paint(w, h)
    -- Drawing code only
end
```

### Remove Panels Properly

```lua
-- Bad: Memory leak
function PLUGIN:ShowMenu()
    local frame = vgui.Create("DFrame")
    -- Frame is never removed!
end

-- Good: Close when done
function PLUGIN:ShowMenu()
    if IsValid(self.frame) then
        self.frame:Remove()
    end

    self.frame = vgui.Create("DFrame")
    self.frame:MakePopup()
end

-- Even better: Use ix.gui system
function PLUGIN:ShowMenu()
    local frame = vgui.Create("DFrame")
    ix.gui["MyMenu"] = frame  -- Automatically managed
end
```

### Optimize Paint Hooks

```lua
-- Bad: Complex calculations in Paint
function PANEL:Paint(w, h)
    for i = 1, 100 do
        local x = math.sin(CurTime() * i) * w
        local y = math.cos(CurTime() * i) * h
        surface.SetDrawColor(255, 255, 255)
        surface.DrawRect(x, y, 5, 5)
    end
end

-- Good: Cache calculated positions
function PANEL:Think()
    -- Calculate once per frame
    self.positions = {}
    local curTime = CurTime()
    for i = 1, 100 do
        self.positions[i] = {
            x = math.sin(curTime * i) * self:GetWide(),
            y = math.cos(curTime * i) * self:GetTall()
        }
    end
end

function PANEL:Paint(w, h)
    surface.SetDrawColor(255, 255, 255)
    for _, pos in ipairs(self.positions) do
        surface.DrawRect(pos.x, pos.y, 5, 5)
    end
end
```

## Security

### Never Trust Client Input

```lua
-- Bad: Client can send anything
concommand.Add("giveadmin", function(client, cmd, args)
    client:SetUserGroup("superadmin")  -- Anyone can become admin!
end)

-- Good: Verify permission
ix.command.Add("GiveAdmin", {
    adminOnly = true,  -- Only admins can run
    arguments = {
        ix.type.player
    },
    OnRun = function(self, client, target)
        -- Additional checks
        if not client:IsSuperAdmin() then
            return "You need superadmin!"
        end

        target:SetUserGroup("admin")
    end
})
```

### Validate All Data

```lua
-- Bad: No validation
function PLUGIN:SetPlayerHealth(client, health)
    client:SetHealth(health)  -- Could be negative or huge!
end

-- Good: Validate ranges
function PLUGIN:SetPlayerHealth(client, health)
    health = math.Clamp(tonumber(health) or 100, 1, client:GetMaxHealth())
    client:SetHealth(health)
end
```

### Sanitize User Input

```lua
-- Bad: Can execute Lua code
local code = client:GetNWString("CustomCode")
RunString(code)  -- CLIENT CAN RUN ANYTHING!

-- Good: Don't execute user code on server
-- If you must, use a sandboxed environment with strict limitations
```

### Check Permissions

```lua
-- Bad: Anyone can use
function PLUGIN:OpenAdminPanel(client)
    -- Open panel
end

-- Good: Check permission
function PLUGIN:OpenAdminPanel(client)
    if not client:IsAdmin() then
        client:Notify("No permission!")
        return
    end
    -- Open panel
end
```

## Plugin Development

### Use PLUGIN Table

```lua
-- Bad: Global functions
function MyFunction()
    -- Global pollution!
end

-- Good: Plugin methods
function PLUGIN:MyFunction()
    -- Contained in plugin
end
```

### Store Plugin Data Properly

```lua
-- Bad: Store in globals
MY_PLUGIN_DATA = {}

-- Good: Use plugin data system
function PLUGIN:SaveData()
    self:SetData({
        savedItems = self.items,
        savedPlayers = self.players
    })
end

function PLUGIN:LoadData()
    local data = self:GetData() or {}
    self.items = data.savedItems or {}
    self.players = data.savedPlayers or {}
end
```

### Initialize Properly

```lua
function PLUGIN:InitializedPlugins()
    -- Called after all plugins loaded
    -- Good place to initialize data that depends on other plugins
    self:LoadData()
end

function PLUGIN:InitializedSchema()
    -- Called after schema loaded
    -- Good place for schema-specific initialization
end
```

### Clean Up Resources

```lua
function PLUGIN:OnPluginUnloaded()
    -- Remove timers
    timer.Remove("MyPluginTimer")

    -- Close UI
    if CLIENT and IsValid(self.menu) then
        self.menu:Remove()
    end

    -- Clean up entities
    if SERVER then
        for _, ent in ipairs(ents.FindByClass("my_custom_entity")) do
            ent:Remove()
        end
    end
end
```

## Error Handling

### Check for Nil Values

```lua
-- Bad: Crashes if character doesn't exist
function PLUGIN:GetPlayerMoney(client)
    return client:GetCharacter():GetMoney()  -- ERROR if no character!
end

-- Good: Check for nil
function PLUGIN:GetPlayerMoney(client)
    local character = client:GetCharacter()
    if not character then
        return 0
    end
    return character:GetMoney()
end

-- Better: Use Guard clauses
function PLUGIN:GetPlayerMoney(client)
    if not IsValid(client) then return 0 end

    local character = client:GetCharacter()
    if not character then return 0 end

    return character:GetMoney()
end
```

### Use pcall for Risky Operations

```lua
-- Bad: Could crash entire plugin
local data = util.JSONToTable(jsonString)  -- Could error!

-- Good: Catch errors
local success, data = pcall(util.JSONToTable, jsonString)
if not success then
    print("Failed to parse JSON:", data)
    data = {}
end
```

### Log Errors Properly

```lua
-- Bad: Silent failure
function PLUGIN:DoSomething()
    local success, err = pcall(riskyFunction)
    -- Error is ignored!
end

-- Good: Log errors
function PLUGIN:DoSomething()
    local success, err = pcall(riskyFunction)
    if not success then
        ErrorNoHalt("Plugin Error: ", err, "\n")
        ix.log.Add(client, "pluginError", err)
    end
end
```

## Documentation

### Comment Complex Code

```lua
-- Good: Explain why, not what
-- Calculate diagonal distance using Pythagorean theorem
-- because we need precise distance for zone detection
local distance = math.sqrt(dx * dx + dy * dy)

-- Bad: State the obvious
-- Calculate distance
local distance = math.sqrt(dx * dx + dy * dy)
```

### Use LDoc Annotations

```lua
--- Gets a character's money amount.
-- This function safely retrieves the money value from a character,
-- returning 0 if the character is invalid or doesn't exist.
-- @realm shared
-- @player client The player whose character to check
-- @treturn number The amount of money, or 0 if no character
function PLUGIN:GetCharacterMoney(client)
    local character = client:GetCharacter()
    return character and character:GetMoney() or 0
end
```

### Document Plugin Structure

```lua
--[[
    Plugin: Advanced Vendors
    Author: YourName
    Version: 1.0.0

    Description:
    This plugin adds advanced vendor functionality including:
    - Dynamic pricing based on stock
    - Vendor reputation system
    - Faction-specific inventory
    - Restocking mechanics

    Dependencies:
    - ix.business (optional, for business integration)

    Configuration:
    - vendorRestockTime: How often vendors restock (default: 3600)
    - vendorPriceMultiplier: Price adjustment factor (default: 1.0)

    Hooks:
    - VendorPurchase(client, vendor, item, price)
    - VendorRestock(vendor)

    Commands:
    - /vendor - Opens vendor editor (admin only)
    - /setvendor - Converts NPC to vendor (admin only)
]]
```

### Keep README Updated

```markdown
# My Plugin

Description of what your plugin does.

## Features

- Feature 1
- Feature 2
- Feature 3

## Installation

1. Download plugin
2. Place in `garrysmod/gamemodes/yourschema/plugins/`
3. Restart server

## Configuration

Edit `sh_plugin.lua` to configure:

```lua
PLUGIN.config = {
    setting1 = value1,
    setting2 = value2
}
```

## Commands

- `/command1` - Description
- `/command2` - Description

## Permissions

- `plugin.permission1` - Description
- `plugin.permission2` - Description

## Known Issues

- Issue 1
- Issue 2
```

## Summary Checklist

- [ ] Use correct realm prefixes (sh_, cl_, sv_)
- [ ] Follow naming conventions (camelCase for local, CamelCase for library)
- [ ] Cache frequently used values
- [ ] Use prepared statements for database queries
- [ ] Validate all client input
- [ ] Check for nil values before accessing
- [ ] Use appropriate hooks for the task
- [ ] Clean up resources (timers, panels, entities)
- [ ] Document complex code
- [ ] Test on both client and server
- [ ] Test with multiple players
- [ ] Check for errors in console
- [ ] Profile performance for expensive operations
- [ ] Follow existing codebase style

## See Also

- [Architecture Overview](architecture.md)
- [Plugin System](plugins/plugin-system.md)
- [Command System](systems/commands.md)
- [Database System](libraries/database.md)
