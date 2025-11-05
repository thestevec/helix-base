# Plugin Conflict Resolution

> **Reference**: `gamemode/core/libs/sh_plugin.lua`, `gamemode/core/sh_util.lua`

Guide for diagnosing and resolving conflicts between Helix plugins, including hook conflicts, load order issues, and namespace collisions.

## ⚠️ Important: Use Helix Plugin System

**Always use Helix's `PLUGIN` table** rather than adding hooks directly with `hook.Add()`. The framework provides:
- Automatic hook registration and caching
- Plugin-scoped hook management
- Load order control
- Built-in conflict detection

**Reference**: `gamemode/core/libs/sh_plugin.lua:78-83`

---

## Understanding Plugin Conflicts

### Types of Conflicts

1. **Hook Conflicts** - Multiple plugins modify same behavior differently
2. **Load Order Issues** - Plugin B needs Plugin A but loads first
3. **Global Variable Collisions** - Plugins use same global names
4. **Network String Conflicts** - Two plugins register same net message
5. **Item/Entity ID Conflicts** - UniqueIDs collide
6. **Command Name Conflicts** - Same command registered twice
7. **Configuration Conflicts** - Plugins change same config values

---

## Hook Conflicts

### How Helix Hooks Work

**Reference**: `gamemode/core/libs/sh_plugin.lua:94-106`

**Helix caches plugin hooks** in `HOOKS_CACHE` table:

```lua
-- Internal structure
HOOKS_CACHE = {
    ["PlayerSpawn"] = {
        [PLUGIN_A] = function() end,
        [PLUGIN_B] = function() end
    }
}
```

**All plugin hooks run** unless one returns a value that stops propagation.

### Hook Return Value Behavior

Different hooks handle return values differently:

#### Type 1: First Non-Nil Stops Propagation

**Example**: `CanPlayerUseCharacter`

```lua
-- Plugin A
function PLUGIN:CanPlayerUseCharacter(client, character)
    if character:GetFaction() == FACTION_ADMIN then
        return false, "Admins can't play"  -- STOPS HERE
    end
    -- No return = continue to next plugin
end

-- Plugin B - NEVER RUNS if Plugin A returned false!
function PLUGIN:CanPlayerUseCharacter(client, character)
    if character:GetLevel() < 5 then
        return false, "Level too low"  -- Won't be checked!
    end
end
```

**Conflict**: Plugin B's check never runs if Plugin A blocks first!

**✅ Solution - Check both conditions**:
```lua
-- Combined in one plugin
function PLUGIN:CanPlayerUseCharacter(client, character)
    if character:GetFaction() == FACTION_ADMIN then
        return false, "Admins can't play"
    end

    if character:GetLevel() < 5 then
        return false, "Level too low"
    end
end
```

#### Type 2: All Hooks Run (No Return Value Matters)

**Example**: `PlayerSpawn`, `PlayerDeath`

```lua
-- Plugin A
function PLUGIN:PlayerSpawn(client)
    client:SetHealth(100)
    return true  -- Return value ignored, all hooks still run
end

-- Plugin B - STILL RUNS!
function PLUGIN:PlayerSpawn(client)
    client:SetArmor(50)  -- This executes even if A returned
end
```

**Conflict**: Both plugins modify spawn, can cause unexpected results

**Conflict example**:
```lua
-- Plugin A: Sets default loadout
function PLUGIN:PlayerLoadout(client)
    client:Give("weapon_physcannon")
end

-- Plugin B: Strips all weapons
function PLUGIN:PlayerSpawn(client)
    timer.Simple(0, function()  -- Runs after loadout
        client:StripWeapons()  -- Removes Plugin A's weapon!
    end)
end
```

**✅ Solution - Coordinate or use single plugin**:
```lua
-- Modify Plugin B to respect loadout
function PLUGIN:PlayerSpawn(client)
    timer.Simple(0, function()
        local weps = client:GetWeapons()
        for _, wep in ipairs(weps) do
            if wep:GetClass() != "weapon_physcannon" then  -- Don't strip important weapons
                client:StripWeapon(wep:GetClass())
            end
        end
    end)
end
```

#### Type 3: Return Modifies Behavior

**Example**: `PlayerCanPickupWeapon`

```lua
-- Plugin A
function PLUGIN:PlayerCanPickupWeapon(client, weapon)
    if weapon:GetClass() == "weapon_crowbar" then
        return true  -- Allow pickup
    end
end

-- Plugin B
function PLUGIN:PlayerCanPickupWeapon(client, weapon)
    if client:GetCharacter():GetClass() != CLASS_SOLDIER then
        return false  -- Block all weapon pickups for non-soldiers!
    end
end
```

**Conflict**: If Plugin B runs first, it blocks everything. Order matters!

**✅ Solution - Check in priority order**:
```lua
-- Single authoritative plugin
function PLUGIN:PlayerCanPickupWeapon(client, weapon)
    local char = client:GetCharacter()

    -- Class restriction takes priority
    if char:GetClass() != CLASS_SOLDIER then
        if weapon:GetClass() == "weapon_crowbar" then
            return true  -- Exception: anyone can pickup crowbar
        end
        return false  -- Block others
    end

    -- Soldiers can pickup anything
end
```

---

## Diagnosing Hook Conflicts

### Method 1: Disable Plugins One by One

**Fastest way to find conflict**:

1. Disable half your plugins:
   ```lua
   -- Server console
   ix_unload pluginname
   ```

2. Test if issue persists
3. If persists, disable other half
4. If gone, re-enable plugins one by one until it breaks
5. Last enabled plugin = conflict source

### Method 2: Hook Call Logging

**Add temporary logging** to see hook execution:

```lua
-- Add to sh_plugin.lua temporarily (dev only!)
local oldRun = hook.Run
function hook.Run(name, ...)
    if name == "PlayerSpawn" then  -- Hook you're debugging
        print("[HOOK] " .. name .. " called")

        -- Log which plugins have this hook
        if HOOKS_CACHE[name] then
            for plugin, func in pairs(HOOKS_CACHE[name]) do
                print("  - Plugin: " .. (plugin.name or "Unknown"))
            end
        end
    end

    return oldRun(name, ...)
end
```

### Method 3: Check Plugin Load Order

**See which plugins loaded**:
```lua
-- Console
ix_plugins

-- Or Lua:
lua_run PrintTable(ix.plugin.list)
```

**Load order**:
1. Helix core plugins (`helix/plugins/`)
2. Schema (`schema/sh_schema.lua`)
3. Schema plugins (`schema/plugins/`)

**Reference**: `gamemode/core/libs/sh_plugin.lua:246-258`

### Method 4: Inspect Specific Hook

**See which plugins use a hook**:
```lua
-- Server console
lua_run if HOOKS_CACHE["PlayerSpawn"] then for p, f in pairs(HOOKS_CACHE["PlayerSpawn"]) do print(p.name or p.uniqueID) end end
```

---

## Common Conflict Scenarios

### Conflict 1: Multiple Death Handlers

**Scenario**: PermaDeath plugin + Custom Respawn plugin

```lua
-- Plugin A: Permadeath
function PLUGIN:PlayerDeath(client, inflictor, attacker)
    client:GetCharacter():Delete()  -- Deletes character
end

-- Plugin B: Custom respawn
function PLUGIN:PlayerDeath(client)
    timer.Simple(5, function()
        if IsValid(client) then
            client:Spawn()  -- Tries to respawn deleted character!
        end
    end)
end
```

**❌ Result**: Errors, broken character

**✅ Solution**:
```lua
-- Plugin B: Check character exists
function PLUGIN:PlayerDeath(client)
    timer.Simple(5, function()
        if IsValid(client) and client:GetCharacter() then  -- Verify character exists
            client:Spawn()
        end
    end)
end
```

### Conflict 2: Network String Collision

**Scenario**: Two plugins use same network message name

```lua
-- Plugin A
if SERVER then
    util.AddNetworkString("UpdateData")
end

-- Plugin B
if SERVER then
    util.AddNetworkString("UpdateData")  -- Same name!
end

-- When client receives "UpdateData", which handler runs?
```

**❌ Result**: Wrong handler processes message, errors or unexpected behavior

**✅ Solution - Namespace your net messages**:
```lua
-- Plugin A
if SERVER then
    util.AddNetworkString("MyPluginA:UpdateData")
end

-- Plugin B
if SERVER then
    util.AddNetworkString("MyPluginB:UpdateData")
end
```

**Better**: Use Helix plugin uniqueID:
```lua
PLUGIN.name = "My Plugin"
PLUGIN.uniqueID = "myplugin"  -- Set this!

if SERVER then
    util.AddNetworkString(PLUGIN.uniqueID .. ":UpdateData")
end
```

### Conflict 3: Item UniqueID Collision

**Scenario**: Two plugins register item with same uniqueID

```lua
-- Plugin A: items/sh_medkit.lua
ITEM.uniqueID = "medkit"
ITEM.name = "Medical Kit"

-- Plugin B: items/sh_medkit.lua
ITEM.uniqueID = "medkit"  -- Same ID!
ITEM.name = "First Aid Kit"
```

**❌ Result**: One item overwrites the other, unpredictable which loads

**✅ Solution - Prefix with plugin name**:
```lua
-- Plugin A
ITEM.uniqueID = "plugina_medkit"

-- Plugin B
ITEM.uniqueID = "pluginb_medkit"
```

### Conflict 4: Command Name Collision

**Scenario**: Two plugins register same command

```lua
-- Plugin A
ix.command.Add("Give", {
    description = "Gives an item",
    OnRun = function(self, client, arguments)
        -- Plugin A behavior
    end
})

-- Plugin B
ix.command.Add("Give", {  -- Same name!
    description = "Gives money",
    OnRun = function(self, client, arguments)
        -- Plugin B behavior - OVERWRITES Plugin A!
    end
})
```

**❌ Result**: Only last registered command works

**✅ Solution - Use different names**:
```lua
-- Plugin A
ix.command.Add("GiveItem", { ... })

-- Plugin B
ix.command.Add("GiveMoney", { ... })
```

### Conflict 5: Global Variable Pollution

**Scenario**: Plugins use same global variable names

```lua
-- Plugin A
function PLUGIN:Initialize()
    TEMP_DATA = {}  -- Global variable!
end

-- Plugin B
function PLUGIN:PlayerSpawn(client)
    TEMP_DATA = client:GetCharacter():GetData()  -- Overwrites Plugin A's table!
end
```

**❌ Result**: Plugin A's data destroyed

**✅ Solution - Use plugin table for storage**:
```lua
-- Plugin A
function PLUGIN:Initialize()
    self.tempData = {}  -- Scoped to this plugin
end

function PLUGIN:SomeFunction()
    self.tempData[key] = value  -- Access via self
end

-- Plugin B
function PLUGIN:PlayerSpawn(client)
    self.tempData = client:GetCharacter():GetData()  -- Separate from Plugin A
end
```

### Conflict 6: Configuration Value Fights

**Scenario**: Plugins set same config to different values

```lua
-- Plugin A
function PLUGIN:InitPostEntity()
    ix.config.Set("maxHealth", 100)
end

-- Plugin B (loads after Plugin A)
function PLUGIN:InitPostEntity()
    ix.config.Set("maxHealth", 150)  -- Overwrites!
end
```

**❌ Result**: Last plugin wins, unpredictable based on load order

**✅ Solution - Create plugin-specific configs**:
```lua
-- Plugin A
function PLUGIN:InitPostEntity()
    ix.config.Add("pluginAMaxHealth", 100, "Max health for Plugin A")
end

-- Plugin B
function PLUGIN:InitPostEntity()
    ix.config.Add("pluginBMaxHealth", 150, "Max health for Plugin B")
end
```

---

## Load Order Issues

### Understanding Load Order

**Reference**: `gamemode/core/libs/sh_plugin.lua:246-258`

**Load sequence**:
1. Helix gamemode core
2. Helix core plugins (`helix/plugins/`)
3. Schema (`schema/sh_schema.lua`)
4. Schema plugins (`schema/plugins/`)

**Within each directory**: Alphabetical by folder name

**Example**:
```
helix/plugins/
  - area/        (loads 1st)
  - doors/       (loads 2nd)
  - vendor/      (loads 3rd)
```

### Dependency Issues

**Problem**: Plugin B needs Plugin A's functions but loads first

```lua
-- Plugin A: utils (loads second due to name)
PLUGIN.name = "Utils"
PLUGIN.uniqueID = "utils"

function PLUGIN:GetSpecialValue()
    return self.specialValue or 0
end

-- Plugin B: main (loads first alphabetically)
PLUGIN.name = "Main"

function PLUGIN:Initialize()
    local utils = ix.plugin.Get("utils")
    -- utils is nil! Hasn't loaded yet!
    local val = utils:GetSpecialValue()  -- ERROR!
end
```

**✅ Solution 1 - Use InitializedPlugins hook**:
```lua
-- Plugin B
function PLUGIN:InitializedPlugins()
    -- All plugins are loaded now
    local utils = ix.plugin.Get("utils")
    if utils then
        local val = utils:GetSpecialValue()  -- Works!
    end
end
```

**✅ Solution 2 - Rename folder to control order**:
```
plugins/
  - 00_utils/     (loads first)
  - main/         (loads second)
```

**✅ Solution 3 - Declare dependency**:
```lua
-- Plugin B
PLUGIN.name = "Main"
PLUGIN.depends = {"utils"}  -- Document dependency

function PLUGIN:Initialize()
    -- Still use InitializedPlugins for safety
end

function PLUGIN:InitializedPlugins()
    local utils = ix.plugin.Get("utils")
    assert(utils, "Plugin 'utils' is required but not loaded!")
end
```

---

## Resolving Conflicts

### Step 1: Identify Conflicting Plugins

**Use process of elimination**:

1. Note the symptoms (error message, unexpected behavior)
2. Disable all custom plugins
3. Re-enable plugins one by one
4. When issue appears, last enabled plugin is involved
5. Keep it enabled, disable others, re-enable one by one
6. Find second conflicting plugin

### Step 2: Examine Hook Implementation

**Compare the hooks**:

```lua
-- In Plugin A folder
-- Find: sh_plugin.lua or sv_plugin.lua

-- Look for hook that matches symptom
-- e.g., if players can't spawn, check PlayerSpawn hook
```

**Look for**:
- Return values that block behavior
- Modifications to same values
- Timer conflicts
- Order-dependent logic

### Step 3: Choose Resolution Strategy

**Option A: Merge plugins** (if you control both)
```lua
-- Combine into single plugin with coordinated logic
```

**Option B: Modify one plugin** (to respect other)
```lua
-- Add checks to avoid conflict
function PLUGIN:PlayerSpawn(client)
    -- Check if other plugin already handled this
    if client.myPluginSpawnHandled then return end

    -- Your logic here
end
```

**Option C: Use plugin dependencies**
```lua
-- Make Plugin B aware of Plugin A
function PLUGIN:InitializedPlugins()
    local pluginA = ix.plugin.Get("plugina")
    if pluginA then
        -- Adjust behavior based on Plugin A presence
    end
end
```

**Option D: Contact plugin authors**
- Report conflict to both plugin developers
- Request compatibility patch
- Contribute fix via pull request

### Step 4: Test Thoroughly

After fix:
1. Test with both plugins enabled
2. Test each plugin's functionality still works
3. Test the specific conflict scenario
4. Test on clean server
5. Test with other plugins

---

## Preventing Conflicts in Your Plugins

### Best Practices

#### 1. Namespace Everything

**✅ DO**:
```lua
-- Network strings
util.AddNetworkString(PLUGIN.uniqueID .. ":Action")

-- Global functions (avoid, but if needed)
_G["MyPlugin_" .. functionName] = function() end

-- Timers
timer.Create(PLUGIN.uniqueID .. ":Think", 1, 0, function() end)

-- Items
ITEM.uniqueID = "myplugin_item_medkit"

-- Commands
ix.command.Add("MyPluginGive", { ... })
```

#### 2. Use Plugin Table for Storage

**❌ DON'T**:
```lua
CACHED_DATA = {}  -- Global pollution!
```

**✅ DO**:
```lua
function PLUGIN:Initialize()
    self.cachedData = {}
end

function PLUGIN:SomeFunction()
    self.cachedData[key] = value
end
```

#### 3. Validate Hook Returns

**❌ DON'T**:
```lua
function PLUGIN:CanPlayerDoThing(client)
    if condition then
        return false  -- Blocks ALL other plugins from allowing it
    end
end
```

**✅ DO**:
```lua
function PLUGIN:CanPlayerDoThing(client)
    if condition then
        return false
    end
    -- Explicitly return nil to allow other plugins to decide
    return
end
```

#### 4. Document Dependencies

```lua
PLUGIN.name = "Advanced Inventory"
PLUGIN.author = "YourName"
PLUGIN.description = "Adds inventory features"

-- Document requirements
PLUGIN.depends = {"base_inventory"}  -- List dependencies
PLUGIN.conflicts = {"old_inventory"}  -- List known conflicts

function PLUGIN:InitializedPlugins()
    -- Check dependencies
    for _, dep in ipairs(self.depends or {}) do
        if not ix.plugin.Get(dep) then
            error(string.format("Plugin '%s' requires '%s' to be loaded!", self.name, dep))
        end
    end

    -- Warn about conflicts
    for _, conflict in ipairs(self.conflicts or {}) do
        if ix.plugin.Get(conflict) then
            print(string.format("[WARNING] Plugin '%s' may conflict with '%s'", self.name, conflict))
        end
    end
end
```

#### 5. Provide Compatibility Hooks

**Allow other plugins to modify your behavior**:

```lua
-- In your plugin
function PLUGIN:GiveStartingItems(client)
    local items = {"item_medkit", "item_flashlight"}

    -- Let other plugins modify item list
    items = hook.Run("MyPluginStartingItems", client, items) or items

    for _, itemID in ipairs(items) do
        client:GetCharacter():GetInventory():Add(itemID)
    end
end

-- In other plugin (using your plugin)
function PLUGIN:MyPluginStartingItems(client, items)
    -- Add custom item to starting items
    table.insert(items, "custom_item_radio")
    return items
end
```

#### 6. Use Config System

**Make behavior configurable**:

```lua
function PLUGIN:InitializedConfig()
    ix.config.Add("myPluginEnabled", true, "Enable my plugin features", function(oldValue, newValue)
        if newValue then
            self:Enable()
        else
            self:Disable()
        end
    end, {
        category = "My Plugin"
    })
end

-- Other plugins can disable features without modifying code
```

---

## Debugging Tools

### Plugin Conflict Detector

**Add to a debug plugin**:

```lua
-- sv_plugin.lua
function PLUGIN:CheckConflicts()
    local conflicts = {}

    -- Check network strings (can't detect directly, so check naming conventions)
    -- Check for duplicate item IDs
    local itemIDs = {}
    for k, v in pairs(ix.item.list) do
        if itemIDs[k] then
            table.insert(conflicts, {
                type = "Item UniqueID",
                id = k,
                plugins = {itemIDs[k], "unknown"}
            })
        end
        itemIDs[k] = true
    end

    -- Check for duplicate commands
    local commands = {}
    for k, v in pairs(ix.command.list) do
        if commands[k] then
            table.insert(conflicts, {
                type = "Command",
                id = k
            })
        end
        commands[k] = true
    end

    -- Report conflicts
    if #conflicts > 0 then
        print("[CONFLICT DETECTOR] Found " .. #conflicts .. " potential conflicts:")
        PrintTable(conflicts)
    else
        print("[CONFLICT DETECTOR] No conflicts detected")
    end
end

-- Run on server start
timer.Simple(5, function()
    PLUGIN:CheckConflicts()
end)
```

### Hook Execution Tracer

**Trace hook execution for debugging**:

```lua
-- sv_plugin.lua
concommand.Add("ix_tracehook", function(client, cmd, args)
    if not IsValid(client) then  -- Server console only
        local hookName = args[1]
        if not hookName then
            print("Usage: ix_tracehook <hookname>")
            return
        end

        print("Tracing hook: " .. hookName)

        local oldRun = hook.Run
        hook.Run = function(name, ...)
            if name == hookName then
                print("[HOOK TRACE] " .. name .. " called")

                if HOOKS_CACHE[name] then
                    for plugin, func in pairs(HOOKS_CACHE[name]) do
                        local ret = {func(plugin, ...)}
                        if #ret > 0 then
                            print("  Plugin '" .. (plugin.name or plugin.uniqueID) .. "' returned:", unpack(ret))
                        else
                            print("  Plugin '" .. (plugin.name or plugin.uniqueID) .. "' returned nothing")
                        end
                    end
                end
            end

            return oldRun(name, ...)
        end

        timer.Simple(30, function()
            hook.Run = oldRun
            print("Hook tracing ended for " .. hookName)
        end)
    end
end)
```

---

## See Also

- [Common Issues](common-issues.md) - General troubleshooting
- [Plugin System](../plugins/plugin-system.md) - Plugin development
- [Performance Issues](performance.md) - Optimization
- [Creating Plugins](../plugins/creating-plugins.md) - Plugin tutorial
