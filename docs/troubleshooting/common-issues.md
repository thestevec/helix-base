# Common Issues and Solutions

> **Reference**: General troubleshooting guide for Helix framework

This guide covers the most frequently encountered issues when developing with Helix and their solutions.

## ⚠️ Important: Check Framework Documentation First

**Always consult Helix's built-in documentation** before implementing workarounds. Most "issues" stem from:
- Not using framework functions correctly
- Misunderstanding realm separation (server vs client)
- Attempting to override built-in systems
- Loading order problems

## Quick Diagnostic Checklist

Before diving into specific issues, check these common problems:

- [ ] Are you on the correct realm (SERVER vs CLIENT)?
- [ ] Is your file prefixed correctly (`sv_`, `cl_`, `sh_`)?
- [ ] Did you wait for the framework to initialize?
- [ ] Are you using `ix.` namespace functions?
- [ ] Did you check the console for errors?
- [ ] Is your database connection working?

---

## 1. Plugin Not Loading

### Symptoms
- Plugin doesn't appear in `ix_plugins` list
- No errors in console
- Plugin hooks not being called

### Common Causes & Solutions

#### Missing or Incorrect File Name

**Problem**: Plugin file must be named `sh_plugin.lua`

**❌ WRONG**:
```
plugins/myplugin/plugin.lua          -- Wrong name
plugins/myplugin/myplugin.lua        -- Wrong name
plugins/myplugin/sh_myplugin.lua     -- Wrong name
```

**✅ CORRECT**:
```
plugins/myplugin/sh_plugin.lua       -- Correct!
```

**Reference**: `gamemode/core/libs/sh_plugin.lua:55`

#### Missing Plugin Metadata

**Problem**: Plugin needs name, description, and author

**❌ WRONG**:
```lua
-- sh_plugin.lua
function PLUGIN:InitPostEntity()
    -- No metadata defined
end
```

**✅ CORRECT**:
```lua
-- sh_plugin.lua
PLUGIN.name = "My Plugin"
PLUGIN.description = "Does something useful"
PLUGIN.author = "Your Name"

function PLUGIN:InitPostEntity()
    -- Now it will load properly
end
```

#### PluginShouldLoad Hook Blocking

**Problem**: Another plugin is blocking loading with hook return

**Check**:
```lua
-- Search your plugins for this:
function PLUGIN:PluginShouldLoad(uniqueID)
    if uniqueID == "yourplugin" then
        return false  -- This prevents loading!
    end
end
```

**Fix**: Remove or modify the blocking hook

---

## 2. "Attempt to Index a Nil Value" Errors

### Most Common: Accessing ix.* Before Initialization

**❌ WRONG**:
```lua
-- In global scope at file top
local myData = ix.data.Get("key")  -- ix.data might not exist yet!
```

**✅ CORRECT**:
```lua
-- Wait for initialization
function PLUGIN:InitPostEntity()
    local myData = ix.data.Get("key")  -- Safe, framework is loaded
end
```

### Accessing Character on Client Before Load

**❌ WRONG**:
```lua
-- cl_plugin.lua
function PLUGIN:Think()
    local char = LocalPlayer():GetCharacter()
    local name = char:GetName()  -- char might be nil!
end
```

**✅ CORRECT**:
```lua
-- cl_plugin.lua
function PLUGIN:Think()
    local char = LocalPlayer():GetCharacter()

    if not char then return end  -- Always validate!

    local name = char:GetName()
end
```

### Accessing Inventory Before Creation

**❌ WRONG**:
```lua
function PLUGIN:PlayerLoadedCharacter(client, character)
    local inv = character:GetInventory()
    inv:Add("item_medkit")  -- inv might not be loaded yet!
end
```

**✅ CORRECT**:
```lua
function PLUGIN:PlayerLoadedCharacter(client, character, currentChar)
    -- Inventory loads asynchronously
    timer.Simple(0, function()
        if IsValid(client) then
            local inv = character:GetInventory()
            if inv then
                inv:Add("item_medkit")
            end
        end
    end)
end
```

**Better approach** - use the character inventory callback:
```lua
function PLUGIN:CharacterInventoryRestored(character)
    local inv = character:GetInventory()
    -- Inventory is guaranteed to exist here
    inv:Add("item_medkit")
end
```

---

## 3. Realm Confusion (Server vs Client)

### Issue: Code Running on Wrong Realm

**Reference**: `gamemode/core/sh_util.lua:32-63`

#### Understanding File Prefixes

- `sv_` - Server only
- `cl_` - Client only
- `sh_` - Shared (both)

**❌ WRONG**:
```lua
-- cl_myplugin.lua
function PLUGIN:PlayerSpawn(client)
    -- This is a SERVER hook, but you're in CLIENT file!
    client:SetHealth(100)  -- Won't work!
end
```

**✅ CORRECT**:
```lua
-- sv_plugin.lua (or sh_plugin.lua with realm check)
function PLUGIN:PlayerSpawn(client)
    client:SetHealth(100)  -- Works!
end
```

#### Using Realm Guards

**✅ Best Practice**:
```lua
-- sh_plugin.lua (shared file)
if SERVER then
    function PLUGIN:PlayerSpawn(client)
        -- Server-only code
    end
else  -- CLIENT
    function PLUGIN:HUDPaint()
        -- Client-only code
    end
end
```

### Common Realm Mistakes

**Mistake 1: Drawing on Server**
```lua
-- ❌ WRONG - Server doesn't render
if SERVER then
    hook.Add("HUDPaint", "MyHUD", function()
        draw.SimpleText("Hello", "Default", 100, 100)  -- Server has no HUD!
    end)
end
```

**Mistake 2: Spawning Entities on Client**
```lua
-- ❌ WRONG - Client can't create server entities
if CLIENT then
    local ent = ents.Create("prop_physics")  -- Won't persist!
    ent:Spawn()
end
```

**Mistake 3: Networking Without Checking Realm**
```lua
-- ❌ WRONG
function PLUGIN:DoSomething()
    net.Start("MyMessage")  -- Crashes if net message doesn't exist on this realm
    net.Send(player)
end
```

**✅ CORRECT**:
```lua
-- sh_plugin.lua
if SERVER then
    util.AddNetworkString("MyMessage")  -- Register on server

    function PLUGIN:DoSomething(player)
        net.Start("MyMessage")
        net.Send(player)
    end
else  -- CLIENT
    net.Receive("MyMessage", function()
        -- Handle message
    end)
end
```

---

## 4. Items Not Appearing or Working

### Item Not Showing in Inventory

**Check 1: Item is registered**
```lua
-- Console command
ix_itemlist

-- Look for your item uniqueID
```

**Check 2: Item file location**

**✅ CORRECT structure**:
```
plugins/myplugin/items/sh_myitem.lua           -- Basic item
plugins/myplugin/items/base/sh_custom.lua      -- Base type
plugins/myplugin/items/weapons/sh_pistol.lua   -- Categorized
```

**Check 3: Item has required fields**

**❌ WRONG**:
```lua
-- items/sh_myitem.lua
ITEM.name = "My Item"
-- Missing other required fields!
```

**✅ CORRECT**:
```lua
-- items/sh_myitem.lua
ITEM.name = "My Item"
ITEM.description = "A test item"
ITEM.model = "models/props_c17/briefcase001a.mdl"
ITEM.width = 1
ITEM.height = 1
```

### Item Functions Not Being Called

**Problem**: Not understanding item method context

**❌ WRONG**:
```lua
ITEM.name = "Medkit"

function PLUGIN:Use(itemInstance)  -- Wrong! This is a plugin hook, not item method
    -- Won't be called when item is used
end
```

**✅ CORRECT**:
```lua
ITEM.name = "Medkit"

function ITEM:OnUse(client)  -- Item method, called on use
    client:SetHealth(100)
    return true  -- Remove item after use
end
```

**Item Base Inheritance Issue**

```lua
-- If your item isn't working, check it has proper base
ITEM.name = "Custom Weapon"
ITEM.base = "base_weapons"  -- Inherits weapon functionality
ITEM.class = "weapon_pistol"
```

---

## 5. Character Functions Not Working

### Character Not Loading

**Check database**:
```lua
-- Run in server console
lua_run_cl print(LocalPlayer():GetCharacter())

-- If nil, character didn't load
```

**Common cause**: Race condition in hooks

**❌ WRONG**:
```lua
function PLUGIN:PlayerLoadedCharacter(client, character)
    -- Trying to access data that loads asynchronously
    local inv = character:GetInventory()
    local money = character:GetMoney()  -- These might not be ready yet!
end
```

**✅ CORRECT**:
```lua
function PLUGIN:CharacterLoaded(characterID)
    -- Wait for full load
    local character = ix.char.loaded[characterID]

    timer.Simple(0.1, function()
        if character then
            local inv = character:GetInventory()
            -- Safe to use
        end
    end)
end
```

### Character Data Not Saving

**❌ WRONG - Don't modify character table directly**:
```lua
local char = client:GetCharacter()
char.myCustomData = "value"  -- Won't save to database!
```

**✅ CORRECT - Use SetData/GetData**:
```lua
local char = client:GetCharacter()
char:SetData("myCustomData", "value")  -- Saves to database

-- Retrieve later
local value = char:GetData("myCustomData", "default")
```

**Reference**: Character data system documentation

---

## 6. Hooks Not Being Called

### Hook Name Typo

**❌ WRONG**:
```lua
function PLUGIN:PlayerInitialSpawn(client)  -- Wrong capitalization!
    -- Never called
end
```

**✅ CORRECT**:
```lua
function PLUGIN:PlayerSpawn(client)  -- Correct name
    -- Called properly
end
```

**Tip**: Check `docs/hooks/` folder for exact hook signatures

### Hook Timing Issues

Some hooks fire before framework initialization:

**❌ WRONG**:
```lua
function PLUGIN:Initialize()
    -- Too early! Helix tables might not exist
    ix.config.Set("value", 100)  -- Might fail
end
```

**✅ CORRECT**:
```lua
function PLUGIN:InitializedPlugins()
    -- All plugins loaded, safe to use framework
    ix.config.Set("value", 100)
end
```

**Common hook order**:
1. `Initialize()` - Very early, be careful
2. `InitializedPlugins()` - Plugins loaded
3. `InitPostEntity()` - Entities exist
4. `LoadData()` - Schema/plugin data loaded

### Return Values Matter

Some hooks stop propagation with return values:

```lua
function PLUGIN:CanPlayerUseCharacter(client, character)
    if character:GetFaction() == FACTION_ADMIN then
        return false, "You can't use admin characters!"
        -- false stops them, message shows reason
    end
end

function PLUGIN:PlayerSpawn(client)
    client:SetHealth(100)
    -- No return value needed, all hooks will run
end
```

---

## 7. Network Messages Failing

### Network String Not Registered

**❌ WRONG**:
```lua
-- sv_plugin.lua
function PLUGIN:PlayerSpawn(client)
    net.Start("MyMessage")  -- Not registered!
    net.Send(client)  -- Error!
end
```

**✅ CORRECT**:
```lua
-- sv_plugin.lua
util.AddNetworkString("MyMessage")  -- Must be on server

function PLUGIN:PlayerSpawn(client)
    net.Start("MyMessage")
    net.Send(client)  -- Works!
end
```

### Sending to Wrong Realm

**Remember**:
- Server can send to clients
- Clients can send to server
- Clients CANNOT send to other clients directly

**❌ WRONG**:
```lua
-- cl_plugin.lua
net.Start("UpdateOtherPlayer")
net.WriteEntity(otherPlayer)
net.Send(otherPlayer)  -- Can't send to other client!
```

**✅ CORRECT**:
```lua
-- cl_plugin.lua
net.Start("UpdateOtherPlayer")
net.WriteEntity(otherPlayer)
net.SendToServer()  -- Send to server

-- sv_plugin.lua
net.Receive("UpdateOtherPlayer", function(len, client)
    local target = net.ReadEntity()
    -- Server relays to target
    net.Start("UpdateReceived")
    net.Send(target)
end)
```

### Helix Networking vs Garry's Mod net library

**⚠️ Prefer Helix networking for framework integration**

**✅ Using ix.net** (recommended for Helix):
```lua
-- Helix handles registration automatically
-- Check gamemode/core/libs/sh_networking.lua
```

**✅ Using net library** (standard Garry's Mod):
```lua
-- Manual registration required
if SERVER then
    util.AddNetworkString("MyMsg")
end

net.Receive("MyMsg", function(len, client)
    -- Handle
end)
```

---

## 8. Console Errors to Understand

### "attempt to call method 'GetCharacter' (a nil value)"

**Cause**: Trying to call method on nil player

**Fix**:
```lua
-- ❌ WRONG
local char = client:GetCharacter()

-- ✅ CORRECT
if IsValid(client) then
    local char = client:GetCharacter()
    if char then
        -- Use character
    end
end
```

### "bad argument #1 to 'pairs' (table expected, got nil)"

**Cause**: Trying to iterate over nil value

**Fix**:
```lua
-- ❌ WRONG
for k, v in pairs(character:GetInventory():GetItems()) do
end

-- ✅ CORRECT
local inv = character:GetInventory()
if inv then
    local items = inv:GetItems()
    if items then
        for k, v in pairs(items) do
            -- Safe
        end
    end
end
```

### "[ERROR] gamemodes/helix/gamemode/core/libs/sh_plugin.lua:55: attempt to index field 'plugin' (a nil value)"

**Cause**: Circular plugin dependency or incorrect plugin structure

**Fix**:
- Check plugin folder structure
- Ensure sh_plugin.lua exists
- Remove circular dependencies between plugins

---

## 9. Database Issues

See [troubleshooting/database.md](database.md) for detailed database troubleshooting.

**Quick checks**:
```lua
-- Check database connection
print(ix.db.config.adapter)  -- Should print "mysqloo" or "sqlite"

-- Test query
ix.db.Query("SELECT 1", function(result)
    print("Database working!")
end)
```

---

## 10. Performance Issues

See [troubleshooting/performance.md](performance.md) for optimization guide.

**Quick fixes**:
- Don't use `Think()` hook unless necessary
- Cache frequently accessed values
- Use `timer.Simple` instead of loops
- Avoid creating tables in render hooks

---

## Getting Help

### Information to Provide

When asking for help, include:

1. **Full error message** from console
2. **Relevant code snippet** (not entire file)
3. **What you tried** to fix it
4. **Realm** (server or client)
5. **Helix version** (`ix.version`)
6. **Related plugins** that might conflict

### Useful Console Commands

```lua
-- Print Helix version
lua_run print(ix.version)

-- List loaded plugins
ix_plugins

-- Show item list
ix_itemlist

-- Test database
lua_run_sv ix.db.Query("SELECT 1", function(r) print(r) end)

-- Check character
lua_run_cl print(LocalPlayer():GetCharacter())

-- Reload plugin (dev mode only)
ix_reloadplugin myplugin
```

### Debug Techniques

**Print debugging**:
```lua
function PLUGIN:PlayerSpawn(client)
    print("PlayerSpawn called for:", client)
    print("Character:", client:GetCharacter())

    if client:GetCharacter() then
        print("Character name:", client:GetCharacter():GetName())
    end
end
```

**Conditional breakpoints**:
```lua
function PLUGIN:Think()
    local char = LocalPlayer():GetCharacter()

    if char and char:GetName() == "John Doe" then
        -- Debug specific character
        print("Debug info for John Doe")
    end
end
```

**Error handling**:
```lua
function PLUGIN:DoSomething()
    local success, err = pcall(function()
        -- Risky code
        local result = riskyFunction()
        return result
    end)

    if not success then
        print("Error occurred:", err)
    end
end
```

---

## See Also

- [Database Troubleshooting](database.md)
- [Plugin Conflicts](conflicts.md)
- [Performance Optimization](performance.md)
- [Character System](../systems/character.md)
- [Plugin Development](../plugins/plugin-system.md)
- [Inventory System](../systems/inventory.md)
