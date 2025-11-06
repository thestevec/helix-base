# Options System (ix.option)

> **Reference**: `gamemode/core/libs/sh_option.lua`

The options system provides client-side configuration management, allowing players to customize their gameplay experience. It's analogous to `ix.config`, but specifically for client-side user preferences that can optionally be synced to the server.

## ⚠️ Important: Use Built-in Helix Options

**Always use Helix's ix.option system** rather than creating custom CVars or configuration systems. The framework provides:
- Automatic data persistence on the client
- Optional network synchronization to server
- Built-in UI integration (options menu)
- Type validation and constraints
- Callback support for value changes
- Categorization for organized menus

## Core Concepts

### What are Options?

Options are client-side configuration values that allow players to customize their experience. Unlike server configs (`ix.config`), options are stored per-client and persist across sessions. Common uses include:
- UI preferences (animation speed, blur effects)
- Display settings (24-hour time, tooltips)
- Gameplay preferences (auto-open bags, language)
- Accessibility options

### Client vs Server Realm

**Client-only options** (default):
- Defined with `if (CLIENT) then` block
- Only stored on the client
- Server cannot access these values

**Networked options**:
- Defined in shared realm (no CLIENT block)
- Set `bNetworked = true` in option data
- Client automatically syncs value to server
- Server can retrieve client's option value

### Language Phrases

Options automatically use language phrases for UI display:
- Option title: `opt[OptionName]` (e.g., `optHeadbob` for key `"headbob"`)
- Option description: `optd[OptionName]` (e.g., `optdHeadbob`)
- Category name: The category string itself (e.g., `"appearance"` → `L("appearance")`)

## Using the Options System

### ix.option.Add

**Reference**: `gamemode/core/libs/sh_option.lua:44`

Registers a new client option with the framework.

```lua
-- Boolean option (client-only)
if (CLIENT) then
    ix.option.Add("cheapBlur", ix.type.bool, false, {
        category = "performance"
    })
end

-- Number option with constraints
if (CLIENT) then
    ix.option.Add("animationScale", ix.type.number, 1, {
        category = "appearance",
        min = 0.3,
        max = 2,
        decimals = 1
    })
end

-- Array option (dropdown selection)
if (CLIENT) then
    ix.option.Add("theme", ix.type.array, "default", {
        category = "appearance",
        populate = function()
            return {
                ["default"] = "Default Theme",
                ["dark"] = "Dark Theme",
                ["light"] = "Light Theme"
            }
        end
    })
end

-- Networked option (server can read it)
ix.option.Add("language", ix.type.array, "english", {
    category = "general",
    bNetworked = true,  -- Server will receive this value
    populate = function()
        local entries = {}
        for k, _ in SortedPairs(ix.lang.stored) do
            entries[k] = k:upper()
        end
        return entries
    end
})
```

**Parameters**:
- `key` (string): Unique identifier for this option
- `optionType` (ixtype): Type from `ix.type` (bool, number, string, array)
- `default` (any): Default value
- `data` (table): Additional configuration (see below)

**Data Table Fields**:
- `category` (string): Category for menu grouping (default: `"misc"`)
- `bNetworked` (bool): Whether server should receive this value (default: false)
- `min` (number): Minimum value for number types (default: 0)
- `max` (number): Maximum value for number types (default: 10)
- `decimals` (number): Decimal precision for number types (default: 0)
- `phrase` (string): Custom UI title phrase (default: `"opt" .. key`)
- `description` (string): Custom tooltip phrase (default: `"optd" .. key`)
- `hidden` (function): Function returning bool to hide from menu
- `populate` (function): Function returning table of entries (required for array types)
- `OnChanged` (function): Callback when value changes

### ix.option.Get

**Client Reference**: `gamemode/core/libs/sh_option.lua:241`
**Server Reference**: `gamemode/core/libs/sh_option.lua:301`

Retrieves an option value. Function signature differs between client and server.

**Client Usage**:
```lua
-- Client: Get local player's option
if (CLIENT) then
    local use24Hour = ix.option.Get("24hourTime", false)
    local animScale = ix.option.Get("animationScale", 1)

    -- Use in rendering code
    if (ix.option.Get("cheapBlur", false)) then
        -- Use cheap blur effect
    else
        -- Use expensive blur
    end
end
```

**Server Usage**:
```lua
-- Server: Get specific player's networked option
if (SERVER) then
    local language = ix.option.Get(client, "language", "english")

    -- Send localized message in player's language
    client:Notify(L2(language, "welcomeMessage"))
end
```

**Parameters**:
- **Client**: `ix.option.Get(key, default)`
  - `key` (string): Option identifier
  - `default` (any): Fallback value if not set
- **Server**: `ix.option.Get(client, key, default)`
  - `client` (Player): Player entity
  - `key` (string): Option identifier (must be networked)
  - `default` (any): Fallback value if not set

### ix.option.Set

**Reference**: `gamemode/core/libs/sh_option.lua:209`

Sets an option value for the local player (client-only).

```lua
if (CLIENT) then
    -- Set option value
    ix.option.Set("cheapBlur", true)
    ix.option.Set("animationScale", 1.5)

    -- Set without saving to disk
    ix.option.Set("tempOption", value, true)  -- bNoSave = true
end
```

**Parameters**:
- `key` (string): Option identifier
- `value` (any): New value to set
- `bNoSave` (bool): If true, don't save to disk (default: false)

**Behavior**:
- Automatically clamps number values to min/max
- Triggers `OnChanged` callback if defined
- Networks to server if `bNetworked = true`
- Saves to disk unless `bNoSave = true`

### ix.option.SetDefault

**Reference**: `gamemode/core/libs/sh_option.lua:121`

Changes the default value for an option (useful for schemas overriding framework defaults).

```lua
-- Schema can override framework defaults
ix.option.SetDefault("language", "french")
ix.option.SetDefault("animationScale", 0.8)
```

### ix.option.GetAll

**Reference**: `gamemode/core/libs/sh_option.lua:162`

Returns all registered options with their properties (not values).

```lua
-- Debug: Print all available options
local options = ix.option.GetAll()
for key, data in pairs(options) do
    print(key, data.type, data.default, data.category)
end
```

### ix.option.GetAllByCategories

**Reference**: `gamemode/core/libs/sh_option.lua:180`

Returns options organized by category.

```lua
local categorized = ix.option.GetAllByCategories(true)  -- true = remove hidden
for category, options in pairs(categorized) do
    print("Category:", category)
    for _, option in ipairs(options) do
        print("  -", option.key, "=", option.default)
    end
end
```

## Complete Examples

### Example 1: Plugin with Custom Options

```lua
-- plugins/myoptions/sh_plugin.lua
PLUGIN.name = "My Options"
PLUGIN.description = "Adds custom gameplay options"

if (CLIENT) then
    ix.option.Add("showHealthNumbers", ix.type.bool, false, {
        category = "appearance",
        OnChanged = function(oldValue, newValue)
            -- Refresh HUD when option changes
            hook.Run("RebuildHUD")
        end
    })

    ix.option.Add("hudScale", ix.type.number, 1, {
        category = "appearance",
        min = 0.5,
        max = 2,
        decimals = 1
    })
end

-- Networked option so server can check it
ix.option.Add("pvpMode", ix.type.bool, false, {
    category = "gameplay",
    bNetworked = true  -- Server needs to know this
})
```

### Example 2: Using Options in HUD

```lua
-- plugins/myhud/cl_hooks.lua
function PLUGIN:HUDPaint()
    local client = LocalPlayer()

    -- Check if player wants to see health numbers
    if (ix.option.Get("showHealthNumbers", false)) then
        local health = client:Health()
        local maxHealth = client:GetMaxHealth()

        -- Get HUD scale from options
        local scale = ix.option.Get("hudScale", 1)

        draw.SimpleText(
            health .. " / " .. maxHealth,
            "ixBigFont",
            ScrW() * 0.5,
            ScrH() * 0.9 * scale,
            color_white,
            TEXT_ALIGN_CENTER
        )
    end
end
```

### Example 3: Server Reading Networked Options

```lua
-- plugins/pvpsystem/sv_hooks.lua
function PLUGIN:PlayerCanAttack(attacker, target)
    -- Check if both players have PvP enabled
    local attackerPvP = ix.option.Get(attacker, "pvpMode", false)
    local targetPvP = ix.option.Get(target, "pvpMode", false)

    if (!attackerPvP or !targetPvP) then
        attacker:Notify("Both players must enable PvP mode!")
        return false
    end
end
```

## Best Practices

### ✅ DO

- Use `ix.option.Add()` in appropriate realm (CLIENT for local-only, shared for networked)
- Create language phrases for option names: `L.optYourOption` and `L.optdYourOption`
- Set `bNetworked = true` only when server needs the value
- Use appropriate categories: `"general"`, `"appearance"`, `"performance"`, `"gameplay"`
- Provide sensible min/max values for number types
- Use `OnChanged` callback for immediate UI updates
- Check options at the point of use, not on initialization

### ❌ DON'T

- Don't create custom ConVars for client preferences - use ix.option instead
- Don't try to `ix.option.Set()` from server - options are client-controlled
- Don't access `ix.option.client` table directly - use `ix.option.Get()`
- Don't network options that server doesn't need (wastes bandwidth)
- Don't forget to define options in shared realm if using `bNetworked = true`
- Don't create options for server-wide settings (use `ix.config` instead)

## Common Patterns

### Pattern 1: Performance Toggle

```lua
-- Add option in client realm
if (CLIENT) then
    ix.option.Add("expensiveEffect", ix.type.bool, true, {
        category = "performance"
    })
end

-- Check before expensive operation
if (CLIENT) then
    function PLUGIN:PostDrawOpaqueRenderables()
        if (!ix.option.Get("expensiveEffect", true)) then
            return  -- Skip expensive rendering
        end

        -- Expensive effect here
    end
end
```

### Pattern 2: Array Selection with Populate

```lua
if (CLIENT) then
    ix.option.Add("crosshairStyle", ix.type.array, "default", {
        category = "appearance",
        populate = function()
            return {
                ["default"] = "Default Crosshair",
                ["dot"] = "Simple Dot",
                ["cross"] = "Cross",
                ["none"] = "No Crosshair"
            }
        end
    })
end
```

### Pattern 3: Dynamic Option Visibility

```lua
if (CLIENT) then
    ix.option.Add("advancedMode", ix.type.bool, false, {
        category = "general"
    })

    ix.option.Add("expertSetting", ix.type.number, 5, {
        category = "general",
        hidden = function()
            -- Only show if advanced mode is enabled
            return !ix.option.Get("advancedMode", false)
        end
    })
end
```

### Pattern 4: OnChanged Callback

```lua
if (CLIENT) then
    ix.option.Add("language", ix.type.array, "english", {
        category = "general",
        bNetworked = true,
        populate = function()
            return ix.lang.stored
        end,
        OnChanged = function(oldValue, newValue)
            -- Reload UI when language changes
            ix.gui.Rebuild()

            -- Notify user
            LocalPlayer():Notify(L("languageChanged"))
        end
    })
end
```

## Common Issues

### Options Not Appearing in Menu

**Cause**: Missing language phrases for option name/description
**Fix**: Add language phrases to your plugin or schema

```lua
-- plugins/myplugin/languages/sh_english.lua
LANGUAGE = {
    optMyOption = "My Option",
    optdMyOption = "This is what my option does",

    -- Also define category if custom
    myCategory = "My Category"
}
```

### Server Can't Read Option

**Cause**: Option not marked as networked or defined only in CLIENT realm
**Fix**: Define in shared realm and set `bNetworked = true`

```lua
-- ❌ WRONG: Client-only option, server can't read
if (CLIENT) then
    ix.option.Add("pvpMode", ix.type.bool, false)
end

-- ✅ CORRECT: Shared realm with networking
ix.option.Add("pvpMode", ix.type.bool, false, {
    category = "gameplay",
    bNetworked = true
})
```

### Option Value Not Persisting

**Cause**: Using `bNoSave = true` or calling `Set` before framework loads
**Fix**: Remove `bNoSave` flag and ensure timing is correct

```lua
-- ❌ WRONG: Won't save
ix.option.Set("myOption", value, true)

-- ✅ CORRECT: Will save to disk
ix.option.Set("myOption", value)
```

### Number Option Ignoring Input

**Cause**: Value exceeds min/max constraints
**Fix**: Set appropriate min/max or adjust constraints

```lua
-- If trying to set value to 5 but max is 2, it will clamp to 2
ix.option.Add("speed", ix.type.number, 1, {
    category = "gameplay",
    min = 0.1,
    max = 10,  -- Increase max to allow higher values
    decimals = 1
})
```

## See Also

- [Configuration System](configuration.md) - Server-wide configuration with `ix.config`
- [Data System](../libraries/data.md) - Low-level key-value storage used by options
- [Language System](../advanced/localization.md) - Creating language phrases for options
- Source: `gamemode/core/libs/sh_option.lua`
- Default Options: `gamemode/config/sh_options.lua`
