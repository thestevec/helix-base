# Performance Optimization

> **Reference**: General performance optimization guide for Helix framework

Guide for diagnosing and fixing performance issues, including server lag, FPS drops, and optimization techniques.

## ⚠️ Important: Profile Before Optimizing

**Always measure performance before and after optimizations**. Don't guess:
- Use `concommand.Run(client, "showbudget")` to see performance breakdown
- Profile specific functions with timing code
- Test with realistic player counts
- Verify optimizations actually help

**Premature optimization is the root of all evil** - Focus on proven bottlenecks first.

---

## Quick Performance Checks

### Server Performance

```lua
-- Check server framerate
lua_run print("Server FPS: " .. math.Round(1 / FrameTime()))
-- Should be close to tickrate (usually 66)

-- Check tick time
lua_run print("Tick time: " .. FrameTime() * 1000 .. "ms")
-- Should be ~15ms for 66 tick

-- Check hook time
showbudget
-- Shows breakdown of CPU time per system
```

### Client Performance

```
cl_showfps 1              // Show FPS
net_graph 1               // Show network stats
showbudget                // Show performance breakdown
r_speeds 1                // Show rendering stats
```

### Common Performance Metrics

- **Server FPS** should be at or near tickrate (66)
- **Server tick time** under 15ms (for 66 tick)
- **Client FPS** 60+ recommended
- **Network ping** under 100ms ideal
- **Entity count** under 2000 recommended

---

## Server Performance Issues

### Issue 1: Low Server FPS

**Symptoms**:
- Server FPS below tickrate
- Lag for all players
- Physics stuttering
- Delayed responses

### Common Causes

#### Cause 1: Too Many Think Hooks

**Think() hooks run every frame** - extremely expensive!

**❌ BAD** - Think hook that runs every frame:
```lua
function PLUGIN:Think()
    for _, client in ipairs(player.GetAll()) do
        -- Runs every frame for every player!
        local char = client:GetCharacter()
        if char then
            char:SetData("lastSeen", CurTime())
        end
    end
end
```

**✅ GOOD** - Use timer instead:
```lua
function PLUGIN:InitializedPlugins()
    timer.Create(self.uniqueID .. ":UpdateSeen", 1, 0, function()
        -- Runs once per second, not 66 times!
        for _, client in ipairs(player.GetAll()) do
            local char = client:GetCharacter()
            if char then
                char:SetData("lastSeen", CurTime())
            end
        end
    end)
end
```

**Better** - Use appropriate hook:
```lua
function PLUGIN:PlayerDisconnected(client)
    -- Only runs when player disconnects
    local char = client:GetCharacter()
    if char then
        char:SetData("lastSeen", CurTime())
    end
end
```

#### Cause 2: Expensive Loops in Hooks

**❌ BAD** - Nested loops in frequently-called hook:
```lua
function PLUGIN:PlayerTick(client)
    -- Runs 66 times per second PER PLAYER!
    for _, ent in ipairs(ents.GetAll()) do  -- Loops through ALL entities
        for _, item in pairs(ix.item.instances) do  -- Loops through ALL items
            if ent:GetPos():Distance(item:GetPos()) < 100 then
                -- Do something
            end
        end
    end
end
```

**✅ GOOD** - Cache and throttle:
```lua
function PLUGIN:InitializedPlugins()
    self.nextCheck = 0
    self.nearbyCache = {}
end

function PLUGIN:Think()
    if CurTime() < self.nextCheck then return end
    self.nextCheck = CurTime() + 1  -- Only run once per second

    -- Process one player at a time per tick
    local players = player.GetAll()
    self.currentPlayerIndex = (self.currentPlayerIndex or 0) + 1
    if self.currentPlayerIndex > #players then
        self.currentPlayerIndex = 1
    end

    local client = players[self.currentPlayerIndex]
    if IsValid(client) then
        -- Process only this player
        local nearby = ents.FindInSphere(client:GetPos(), 100)
        self.nearbyCache[client] = nearby
    end
end
```

#### Cause 3: Database Queries in Loops

**❌ BAD** - Query database repeatedly:
```lua
function PLUGIN:SomeFunction()
    for _, client in ipairs(player.GetAll()) do
        -- Separate query for each player!
        ix.db.Query("SELECT * FROM ix_players WHERE steamid = '" .. client:SteamID() .. "'", function(result)
            -- Process
        end)
    end
end
```

**✅ GOOD** - Batch query:
```lua
function PLUGIN:SomeFunction()
    local steamIDs = {}
    for _, client in ipairs(player.GetAll()) do
        table.insert(steamIDs, mysql:Escape(client:SteamID()))
    end

    -- Single query for all players
    local query = mysql:Select("ix_players")
        query:WhereIn("steamid", steamIDs)
        query:Callback(function(result)
            -- Process all results
        end)
    query:Execute()
end
```

#### Cause 4: Too Many Entities

**Check entity count**:
```lua
lua_run print("Entity count: " .. #ents.GetAll())
```

**If over 2000**, consider:
- Removing unused props
- Using instanced props where possible
- Cleaning up old entities
- Limiting prop spawn

**Clean up old entities**:
```lua
-- Find entities that haven't been used
for _, ent in ipairs(ents.FindByClass("prop_physics")) do
    if not ent.lastUsedTime then
        ent.lastUsedTime = CurTime()
    end

    -- Remove props unused for 1 hour
    if CurTime() - ent.lastUsedTime > 3600 then
        ent:Remove()
    end
end
```

---

## Client Performance Issues

### Issue 1: Low FPS

**Symptoms**:
- Client FPS drops below 60
- Stuttering
- Input lag
- Rendering issues

### Common Causes

#### Cause 1: HUDPaint Doing Too Much

**HUDPaint runs every frame** - must be fast!

**❌ BAD**:
```lua
function PLUGIN:HUDPaint()
    -- Expensive operations every frame!
    local char = LocalPlayer():GetCharacter()

    for k, v in pairs(ix.item.instances) do  -- Loops through ALL items
        if v:GetOwner() == char then
            -- Draw item info
            draw.SimpleText(v:GetName(), "Default", ScrW() / 2, 100)
        end
    end

    -- Drawing too many things
    for i = 1, 1000 do
        draw.RoundedBox(4, i * 10, 100, 8, 8, Color(255, 0, 0))
    end
end
```

**✅ GOOD** - Cache and minimize:
```lua
function PLUGIN:InitializedPlugins()
    self.cachedItems = {}
    self.nextUpdate = 0
end

function PLUGIN:HUDPaint()
    -- Update cache periodically, not every frame
    if CurTime() > self.nextUpdate then
        self.nextUpdate = CurTime() + 0.5
        self:UpdateItemCache()
    end

    -- Only draw cached data
    if self.cachedItems then
        for i, itemName in ipairs(self.cachedItems) do
            draw.SimpleText(itemName, "Default", ScrW() / 2, 100 + (i * 20))
        end
    end
end

function PLUGIN:UpdateItemCache()
    self.cachedItems = {}
    local char = LocalPlayer():GetCharacter()

    if char then
        local inv = char:GetInventory()
        if inv then
            local items = inv:GetItems()
            for k, v in pairs(items) do
                table.insert(self.cachedItems, v:GetName())
            end
        end
    end
end
```

#### Cause 2: Creating Materials/Fonts Every Frame

**❌ BAD** - Creating resources in render hooks:
```lua
function PLUGIN:HUDPaint()
    -- NEVER do this! Creates new material every frame!
    local mat = Material("materials/myimage.png")
    surface.SetMaterial(mat)
    surface.DrawTexturedRect(0, 0, 100, 100)

    -- NEVER do this! Creates font every frame!
    surface.CreateFont("MyFont", {
        font = "Arial",
        size = 24
    })
end
```

**✅ GOOD** - Create once in Initialize:
```lua
function PLUGIN:InitializedPlugins()
    self.myMaterial = Material("materials/myimage.png")

    surface.CreateFont("MyPluginFont", {
        font = "Arial",
        size = 24,
        weight = 500
    })
end

function PLUGIN:HUDPaint()
    surface.SetMaterial(self.myMaterial)
    surface.DrawTexturedRect(0, 0, 100, 100)

    draw.SimpleText("Text", "MyPluginFont", 100, 100)
end
```

#### Cause 3: Too Many Network Receives

**❌ BAD** - Sending too much data to clients:
```lua
-- Server sends every tick
function PLUGIN:Think()
    net.Start("UpdatePositions")
    for _, client in ipairs(player.GetAll()) do
        net.WriteVector(client:GetPos())
    end
    net.Broadcast()  -- Sends to all clients every tick!
end
```

**✅ GOOD** - Throttle and delta compress:
```lua
-- Server sends only when changed
function PLUGIN:Think()
    for _, client in ipairs(player.GetAll()) do
        local pos = client:GetPos()
        local lastPos = client.lastNetPos or Vector()

        -- Only send if moved significantly
        if pos:Distance(lastPos) > 10 then
            net.Start("UpdatePosition")
                net.WriteEntity(client)
                net.WriteVector(pos)
            net.Broadcast()

            client.lastNetPos = pos
        end
    end
end
```

---

## Network Performance

### Issue 1: High Bandwidth Usage

**Check network usage**:
```
net_graph 1   // Client - shows kb/s in/out
```

### Optimization Techniques

#### Compress Data

**❌ BAD** - Sending unnecessary data:
```lua
net.Start("UpdatePlayer")
    net.WriteTable({
        name = "John Doe",
        health = 100,
        armor = 50,
        position = Vector(0, 0, 0),
        angle = Angle(0, 0, 0),
        velocity = Vector(0, 0, 0),
        -- Tons of data!
    })
net.Send(client)
```

**✅ GOOD** - Send only what changed:
```lua
-- Instead of full table, use specific writes
net.Start("UpdatePlayerHealth")
    net.WriteUInt(100, 10)  -- 0-1023 range, uses 10 bits instead of 32
net.Send(client)

-- Use appropriate data types
net.WriteUInt(value, 8)    // For 0-255 (1 byte)
net.WriteInt(value, 16)    // For -32768 to 32767 (2 bytes)
net.WriteFloat(value)      // For decimals (4 bytes)
net.WriteDouble(value)     // For high precision (8 bytes)
```

#### Batch Network Messages

**❌ BAD** - Multiple small messages:
```lua
for _, client in ipairs(player.GetAll()) do
    net.Start("UpdateHealth")
        net.WriteUInt(client:Health(), 10)
    net.Broadcast()  // Separate message for each player!
end
```

**✅ GOOD** - Batch into one message:
```lua
net.Start("UpdateAllHealth")
    net.WriteUInt(#player.GetAll(), 7)  // Player count
    for _, client in ipairs(player.GetAll()) do
        net.WriteEntity(client)
        net.WriteUInt(client:Health(), 10)
    end
net.Broadcast()  // Single message with all data
```

#### Send Only to Relevant Clients

**❌ BAD** - Broadcast everything:
```lua
net.Start("ItemPickedUp")
    net.WriteString("John picked up medkit")
net.Broadcast()  // Everyone sees, even if far away
```

**✅ GOOD** - Send to nearby only:
```lua
local recipients = {}
for _, ply in ipairs(player.GetAll()) do
    if ply:GetPos():Distance(client:GetPos()) < 1000 then
        table.insert(recipients, ply)
    end
end

net.Start("ItemPickedUp")
    net.WriteEntity(client)
    net.WriteString("medkit")
net.Send(recipients)  // Only nearby players
```

---

## Database Performance

### Issue 1: Slow Queries

See [Database Troubleshooting](database.md) for detailed guide.

**Quick tips**:

#### Use Indexes

```sql
-- Add index for frequently queried columns
ALTER TABLE ix_characters ADD INDEX idx_faction (faction);
ALTER TABLE ix_items ADD INDEX idx_inventory (inventory_id);
```

#### Limit Results

**❌ BAD**:
```lua
ix.db.Query("SELECT * FROM ix_characters")  -- Loads ALL characters!
```

**✅ GOOD**:
```lua
ix.db.Query("SELECT * FROM ix_characters LIMIT 100")  -- Limit results
```

#### Select Only Needed Columns

**❌ BAD**:
```lua
ix.db.Query("SELECT * FROM ix_characters")  -- Loads all columns
```

**✅ GOOD**:
```lua
ix.db.Query("SELECT id, name, faction FROM ix_characters")  -- Only needed columns
```

---

## Memory Optimization

### Issue 1: Memory Leaks

**Check memory usage**:
```lua
-- Server console
lua_run collectgarbage("count")  // Returns KB used
```

**If memory constantly increases**, you have a leak.

### Common Leak Causes

#### Cause 1: Timers Not Removed

**❌ BAD**:
```lua
function PLUGIN:PlayerSpawn(client)
    -- Creates new timer every spawn!
    timer.Create("CheckPlayer", 1, 0, function()
        if IsValid(client) then
            -- Check player
        end
    end)
end
-- Timer never removed, stacks up!
```

**✅ GOOD**:
```lua
function PLUGIN:PlayerSpawn(client)
    local timerName = "CheckPlayer_" .. client:SteamID()

    -- Remove old timer if exists
    timer.Remove(timerName)

    timer.Create(timerName, 1, 0, function()
        if not IsValid(client) then
            timer.Remove(timerName)  // Clean up
            return
        end
        -- Check player
    end)
end

function PLUGIN:PlayerDisconnected(client)
    -- Clean up timer
    timer.Remove("CheckPlayer_" .. client:SteamID())
end
```

#### Cause 2: Tables Never Cleaned

**❌ BAD**:
```lua
PLUGIN.playerData = {}

function PLUGIN:PlayerSpawn(client)
    self.playerData[client] = {
        spawnTime = CurTime(),
        -- Data
    }
    -- Never cleaned when player leaves!
end
```

**✅ GOOD**:
```lua
PLUGIN.playerData = {}

function PLUGIN:PlayerSpawn(client)
    self.playerData[client] = {
        spawnTime = CurTime()
    }
end

function PLUGIN:PlayerDisconnected(client)
    self.playerData[client] = nil  // Clean up!
end
```

#### Cause 3: Hooks Not Removed

**❌ BAD**:
```lua
function PLUGIN:SomeEvent()
    hook.Add("Think", "MyThinkHook", function()
        -- Do something
    end)
    -- Hook never removed!
end
```

**✅ GOOD**:
```lua
function PLUGIN:SomeEvent()
    local hookName = "MyThinkHook_" .. CurTime()

    hook.Add("Think", hookName, function()
        -- Do something once
        hook.Remove("Think", hookName)  // Remove after use
    end)
end
```

---

## Profiling and Measurement

### Basic Profiling

**Time a function**:
```lua
local startTime = SysTime()

-- Code to measure
for i = 1, 1000 do
    -- Do something
end

local duration = (SysTime() - startTime) * 1000
print("Operation took: " .. duration .. "ms")
```

### Profile Hook Performance

**Add profiling to hook**:
```lua
local oldHook = PLUGIN.PlayerTick
function PLUGIN:PlayerTick(client)
    local start = SysTime()

    oldHook(self, client)

    local duration = (SysTime() - start) * 1000
    if duration > 1 then  -- Log if over 1ms
        print(string.format("[PROFILE] PlayerTick took %.2fms", duration))
    end
end
```

### Automated Performance Monitoring

**Create monitoring plugin**:
```lua
-- sv_plugin.lua
PLUGIN.name = "Performance Monitor"

function PLUGIN:Initialize()
    self.hookTimes = {}
end

function PLUGIN:Think()
    if not self.nextReport or CurTime() > self.nextReport then
        self.nextReport = CurTime() + 10

        -- Report slowest hooks
        local sorted = {}
        for hook, time in pairs(self.hookTimes) do
            table.insert(sorted, {hook = hook, time = time})
        end

        table.SortByMember(sorted, "time", false)

        print("[PERFORMANCE] Slowest hooks:")
        for i = 1, math.min(5, #sorted) do
            print(string.format("  %s: %.2fms", sorted[i].hook, sorted[i].time))
        end

        self.hookTimes = {}
    end
end
```

### Client FPS Monitoring

**Track client performance**:
```lua
-- cl_plugin.lua
function PLUGIN:Initialize()
    self.fpsHistory = {}
end

function PLUGIN:HUDPaint()
    local fps = math.Round(1 / FrameTime())
    table.insert(self.fpsHistory, fps)

    if #self.fpsHistory > 100 then
        table.remove(self.fpsHistory, 1)
    end

    -- Calculate average
    local total = 0
    for _, f in ipairs(self.fpsHistory) do
        total = total + f
    end
    local avg = total / #self.fpsHistory

    -- Warn if low
    if avg < 30 then
        draw.SimpleText("WARNING: Low FPS (" .. math.Round(avg) .. ")", "Default", 10, 10, Color(255, 0, 0))
    end
end
```

---

## Optimization Best Practices

### General Rules

1. **Cache frequently accessed values**
   ```lua
   -- ❌ BAD
   for i = 1, 100 do
       local char = LocalPlayer():GetCharacter()  -- Calls function 100 times
   end

   -- ✅ GOOD
   local char = LocalPlayer():GetCharacter()  -- Call once
   for i = 1, 100 do
       -- Use cached char
   end
   ```

2. **Use local variables**
   ```lua
   -- ❌ BAD
   for i = 1, 10000 do
       table.insert(_G.myTable, i)  -- Global lookup every iteration
   end

   -- ✅ GOOD
   local myTable = myTable  -- Cache global as local
   for i = 1, 10000 do
       table.insert(myTable, i)  -- Local access is faster
   end
   ```

3. **Avoid string concatenation in loops**
   ```lua
   -- ❌ BAD
   local str = ""
   for i = 1, 1000 do
       str = str .. i  -- Creates new string each iteration
   end

   -- ✅ GOOD
   local parts = {}
   for i = 1, 1000 do
       parts[#parts + 1] = i  -- Append to table
   end
   local str = table.concat(parts)  -- Concat once at end
   ```

4. **Use appropriate data structures**
   ```lua
   -- ❌ BAD - Using table as list for lookups
   local bannedIDs = {"STEAM_0:1:123", "STEAM_0:1:456", ...}
   for _, id in ipairs(bannedIDs) do
       if id == checkID then  -- O(n) lookup
           return true
       end
   end

   -- ✅ GOOD - Using table as hash map
   local bannedIDs = {
       ["STEAM_0:1:123"] = true,
       ["STEAM_0:1:456"] = true
   }
   if bannedIDs[checkID] then  -- O(1) lookup
       return true
   end
   ```

5. **Minimize garbage collection**
   ```lua
   -- ❌ BAD - Creates new table every call
   function PLUGIN:GetPlayerInfo(client)
       return {
           name = client:GetName(),
           health = client:Health()
       }
   end

   -- ✅ GOOD - Reuse table
   function PLUGIN:Initialize()
       self.infoTable = {}
   end

   function PLUGIN:GetPlayerInfo(client)
       local t = self.infoTable
       t.name = client:GetName()
       t.health = client:Health()
       return t
   end
   ```

---

## See Also

- [Common Issues](common-issues.md) - General troubleshooting
- [Database Troubleshooting](database.md) - Database optimization
- [Plugin Conflicts](conflicts.md) - Resolving conflicts
- [Plugin Best Practices](../plugins/best-practices.md) - Plugin development
