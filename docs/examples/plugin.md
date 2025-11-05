# Complete Plugin Example

> **Reference**: `plugins/stamina/sh_plugin.lua`, `plugins/vendor/sh_plugin.lua`

This document provides a complete, working example of a Helix plugin from start to finish.

## ⚠️ Important: Use Built-in Helix Plugin System

**Always use Helix's plugin system** rather than creating standalone Lua files. The framework provides:
- Automatic file loading from plugin directories
- Hook registration and caching
- Data persistence with `SetData()` and `GetData()`
- Integration with all framework systems
- Plugin dependency management

## Complete Example: Point Tracker Plugin

This example creates a full-featured plugin that tracks player points with commands, data persistence, and UI integration.

### Plugin Structure

```
plugins/pointtracker/
├── sh_plugin.lua        # Main plugin file
├── sh_commands.lua      # Commands
├── sv_hooks.lua         # Server-side hooks
└── cl_hooks.lua         # Client-side HUD display
```

### Step 1: Main Plugin File

**File**: `plugins/pointtracker/sh_plugin.lua`

```lua
PLUGIN.name = "Point Tracker"
PLUGIN.author = "Your Name"
PLUGIN.description = "Tracks player points for achievements and rewards"
PLUGIN.version = "1.0.0"

-- Configuration options
ix.config.Add("pointsOnKill", 10, "Points awarded for killing another player", nil, {
    data = {min = 0, max = 1000},
    category = "Point Tracker"
})

ix.config.Add("pointsOnDeath", -5, "Points lost on death", nil, {
    data = {min = -100, max = 0},
    category = "Point Tracker"
})

ix.config.Add("enablePointSystem", true, "Enable the point tracking system", nil, {
    category = "Point Tracker"
})

-- Load additional files
ix.util.Include("sh_commands.lua")
ix.util.Include("sv_hooks.lua")
ix.util.Include("cl_hooks.lua")

-- Server-side initialization
if SERVER then
    function PLUGIN:InitializedPlugins()
        -- Load saved data
        local data = self:GetData()
        self.points = data.points or {}

        print("[Point Tracker] Loaded " .. table.Count(self.points) .. " player records")
    end

    function PLUGIN:SavePoints()
        -- Save data to disk
        self:SetData({
            points = self.points,
            lastSave = os.time()
        })
    end

    function PLUGIN:AddPoints(client, amount, reason)
        if !ix.config.Get("enablePointSystem", true) then
            return
        end

        if !IsValid(client) then
            return
        end

        local character = client:GetCharacter()
        if !character then
            return
        end

        local steamID = client:SteamID()
        self.points[steamID] = (self.points[steamID] or 0) + amount

        -- Network to client
        net.Start("ixPointsUpdate")
            net.WriteInt(self.points[steamID], 32)
        net.Send(client)

        -- Notify player
        if amount > 0 then
            client:Notify("You gained " .. amount .. " points! (" .. reason .. ")")
        elseif amount < 0 then
            client:Notify("You lost " .. math.abs(amount) .. " points. (" .. reason .. ")")
        end

        -- Save periodically
        self:SavePoints()

        return self.points[steamID]
    end

    function PLUGIN:GetPoints(client)
        if !IsValid(client) then
            return 0
        end

        return self.points[client:SteamID()] or 0
    end

    function PLUGIN:OnPluginUnloaded()
        -- Save data before unloading
        self:SavePoints()
        print("[Point Tracker] Plugin unloaded, data saved")
    end
end

-- Client-side initialization
if CLIENT then
    -- Network string for receiving point updates
    net.Receive("ixPointsUpdate", function()
        local points = net.ReadInt(32)
        PLUGIN.clientPoints = points
    end)

    function PLUGIN:InitializedPlugins()
        -- Request initial points from server
        net.Start("ixPointsRequest")
        net.SendToServer()
    end
end
```

### Step 2: Commands

**File**: `plugins/pointtracker/sh_commands.lua`

```lua
-- Command to check points
ix.command.Add("Points", {
    description = "Check your current points",
    OnRun = function(self, client)
        if SERVER then
            local points = PLUGIN:GetPoints(client)
            return "You have " .. points .. " points"
        end
    end
})

-- Admin command to give points
ix.command.Add("GivePoints", {
    description = "Give points to a player",
    adminOnly = true,
    arguments = {
        ix.type.player,
        ix.type.number,
        bit.bor(ix.type.string, ix.type.optional)
    },
    OnRun = function(self, client, target, amount, reason)
        if SERVER then
            reason = reason or "Admin award"
            PLUGIN:AddPoints(target, amount, reason)

            return "Gave " .. amount .. " points to " .. target:Name()
        end
    end
})

-- Admin command to set points
ix.command.Add("SetPoints", {
    description = "Set a player's points",
    adminOnly = true,
    arguments = {
        ix.type.player,
        ix.type.number
    },
    OnRun = function(self, client, target, amount)
        if SERVER then
            local steamID = target:SteamID()
            PLUGIN.points[steamID] = amount

            net.Start("ixPointsUpdate")
                net.WriteInt(amount, 32)
            net.Send(target)

            target:Notify("Your points have been set to " .. amount)

            return "Set " .. target:Name() .. "'s points to " .. amount
        end
    end
})

-- Command to view leaderboard
ix.command.Add("PointsTop", {
    description = "View top 10 players by points",
    OnRun = function(self, client)
        if SERVER then
            -- Sort players by points
            local sorted = {}

            for steamID, points in pairs(PLUGIN.points) do
                table.insert(sorted, {steamID = steamID, points = points})
            end

            table.sort(sorted, function(a, b)
                return a.points > b.points
            end)

            -- Display top 10
            client:ChatPrint("=== Top 10 Players ===")

            for i = 1, math.min(10, #sorted) do
                local data = sorted[i]
                local name = data.steamID -- In real plugin, look up name from database

                -- Try to find online player
                for _, ply in ipairs(player.GetAll()) do
                    if ply:SteamID() == data.steamID then
                        name = ply:Name()
                        break
                    end
                end

                client:ChatPrint(i .. ". " .. name .. ": " .. data.points .. " points")
            end
        end
    end
})
```

### Step 3: Server Hooks

**File**: `plugins/pointtracker/sv_hooks.lua`

```lua
-- Network strings
util.AddNetworkString("ixPointsUpdate")
util.AddNetworkString("ixPointsRequest")

-- Handle point requests from clients
net.Receive("ixPointsRequest", function(length, client)
    local points = PLUGIN:GetPoints(client)

    net.Start("ixPointsUpdate")
        net.WriteInt(points, 32)
    net.Send(client)
end)

-- Award points on kill
function PLUGIN:PlayerDeath(victim, inflictor, attacker)
    if !ix.config.Get("enablePointSystem", true) then
        return
    end

    -- Victim loses points
    if IsValid(victim) and victim:IsPlayer() then
        local loseAmount = ix.config.Get("pointsOnDeath", -5)
        self:AddPoints(victim, loseAmount, "Death")
    end

    -- Attacker gains points
    if IsValid(attacker) and attacker:IsPlayer() and attacker != victim then
        local gainAmount = ix.config.Get("pointsOnKill", 10)
        self:AddPoints(attacker, gainAmount, "Kill")
    end
end

-- Load player points when they spawn
function PLUGIN:PlayerLoadedCharacter(client, character, lastChar)
    -- Send points to client
    timer.Simple(1, function()
        if IsValid(client) then
            local points = self:GetPoints(client)

            net.Start("ixPointsUpdate")
                net.WriteInt(points, 32)
            net.Send(client)
        end
    end)
end

-- Save points periodically
function PLUGIN:InitializedPlugins()
    -- Auto-save every 5 minutes
    timer.Create("ixPointsAutoSave", 300, 0, function()
        PLUGIN:SavePoints()
        print("[Point Tracker] Auto-saved player points")
    end)
end

-- Clean up timer on unload
function PLUGIN:OnPluginUnloaded()
    timer.Remove("ixPointsAutoSave")
end
```

### Step 4: Client HUD Display

**File**: `plugins/pointtracker/cl_hooks.lua`

```lua
-- Store client points locally
PLUGIN.clientPoints = 0

-- Draw points on HUD
function PLUGIN:HUDPaint()
    local client = LocalPlayer()

    if !IsValid(client) or !client:GetCharacter() then
        return
    end

    -- Position in top-right corner
    local scrW, scrH = ScrW(), ScrH()
    local x = scrW - 150
    local y = 50

    -- Draw background
    surface.SetDrawColor(0, 0, 0, 200)
    surface.DrawRect(x - 10, y - 5, 140, 30)

    -- Draw points
    draw.SimpleText("Points: " .. (self.clientPoints or 0), "ixMediumFont", x, y, Color(255, 215, 0), TEXT_ALIGN_LEFT, TEXT_ALIGN_TOP)
end
```

## Using the Plugin

### Installation

1. Create directory: `plugins/pointtracker/`
2. Add the four files above
3. Restart server or reload schema

### Testing Commands

```
!points              - Check your points
!givepoints player 50 "Good job"   - Give 50 points to player (admin)
!setpoints player 100               - Set player points to 100 (admin)
!pointstop           - View leaderboard
```

### Expected Behavior

- Players start with 0 points
- Killing another player: +10 points
- Dying: -5 points
- Points display in top-right corner
- Points persist across server restarts
- Admin can manage points via commands

## ⚠️ Do NOT

```lua
-- WRONG: Don't create global variables
PointData = {}  -- Pollutes global namespace!

-- WRONG: Don't bypass plugin system
hook.Add("PlayerDeath", "MyPointHook", function()
    -- Use PLUGIN:PlayerDeath() instead!
end)

-- WRONG: Don't use file.Write for data
file.Write("points.txt", util.TableToJSON(points))
-- Use self:SetData() instead!

-- WRONG: Don't forget realm checks
function PLUGIN:SomeFunction()
    client:ChatPrint("Hello")  -- Will error on server!
end

-- Correct:
if CLIENT then
    function PLUGIN:SomeFunction()
        chat.AddText("Hello")
    end
end
```

## Best Practices Demonstrated

### ✅ DO

- **Use `self` to access plugin instance**: `self.points`, `self:GetPoints()`
- **Check `IsValid()` before using entities**: Prevents errors
- **Use configuration system**: `ix.config.Add()` for admin control
- **Network data properly**: Use `net` library for client-server sync
- **Save data with plugin methods**: `self:SetData()` and `self:GetData()`
- **Clean up resources**: Remove timers in `OnPluginUnloaded()`
- **Check realms**: Use `if SERVER` and `if CLIENT`
- **Validate input**: Check for nil characters, valid players
- **Use built-in types**: `ix.type.player`, `ix.type.number` in commands

### ❌ DON'T

- Don't use global variables for plugin data
- Don't forget to register network strings
- Don't access client from server without checks
- Don't create memory leaks (timers, panels, hooks)
- Don't trust client input without validation
- Don't modify core framework files
- Don't bypass framework systems

## Advanced Enhancements

### Add Character-Specific Points

```lua
-- Store points per character instead of per player
function PLUGIN:AddPoints(client, amount, reason)
    local character = client:GetCharacter()
    if !character then return end

    local charID = character:GetID()
    self.points[charID] = (self.points[charID] or 0) + amount

    character:SetData("points", self.points[charID])
end
```

### Add Database Storage

```lua
-- Save points to database
function PLUGIN:SaveToDatabase(characterID, points)
    local query = mysql:Update("ix_characters")
    query:Update("data", util.TableToJSON({points = points}))
    query:Where("id", characterID)
    query:Execute()
end
```

### Add Point Rewards

```lua
-- Give rewards at point milestones
function PLUGIN:CheckMilestones(client, points)
    if points >= 100 and !client:GetCharacter():GetData("milestone100") then
        client:GetCharacter():SetData("milestone100", true)
        client:Notify("Milestone reached! You earned a reward!")

        local inv = client:GetCharacter():GetInventory()
        inv:Add("item_medkit", 1)
    end
end
```

## See Also

- [Plugin System](../plugins/plugin-system.md) - Complete plugin documentation
- [Command System](../systems/commands.md) - Creating commands
- [Configuration System](../systems/configuration.md) - Config options
- [Networking](../libraries/networking.md) - Client-server communication
- [Character System](../systems/character.md) - Working with characters
- Source: `plugins/stamina/sh_plugin.lua` - Real plugin example
- Source: `plugins/vendor/sh_plugin.lua` - Complex plugin example
