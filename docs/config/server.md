# Server Configuration

> **Reference**: `gamemode/config/sh_config.lua:145-169`

Server-wide configuration options that control general server behavior, voice chat, weapon mechanics, and business systems.

## ⚠️ Important: Use Built-in Config System

**Always use `ix.config.Add()` and `ix.config.Get()`** rather than creating custom config files. The framework provides:
- Automatic database persistence
- Admin UI for changing values (F1 menu → Config tab)
- Network synchronization to clients
- Type validation and constraints
- Callbacks when values change

## Available Server Options

### allowVoice

**Reference**: `gamemode/config/sh_config.lua:145`

```lua
ix.config.Add("allowVoice", false, "Whether or not voice chat is allowed.", function(oldValue, newValue)
    if (SERVER) then
        hook.Run("VoiceToggled", newValue)
    end
end, {
    category = "server"
})
```

**Type**: Boolean
**Default**: `false`
**Description**: Enables or disables voice chat on the server

**Usage**:
```lua
-- Check if voice is enabled
if ix.config.Get("allowVoice") then
    -- Voice chat is enabled
end
```

**Callback**: Triggers `VoiceToggled` hook when changed, allowing plugins to respond to voice state changes.

---

### voiceDistance

**Reference**: `gamemode/config/sh_config.lua:152`

```lua
ix.config.Add("voiceDistance", 600.0, "How far can the voice be heard.", function(oldValue, newValue)
    if (SERVER) then
        hook.Run("VoiceDistanceChanged", newValue)
    end
end, {
    category = "server",
    data = {min = 0, max = 5000, decimals = 1}
})
```

**Type**: Number (Float)
**Default**: `600.0`
**Range**: 0 - 5000 units
**Description**: Maximum distance in Hammer units that voice chat can be heard

**Usage**:
```lua
-- Get current voice distance
local distance = ix.config.Get("voiceDistance", 600)
```

**Callback**: Triggers `VoiceDistanceChanged` hook when changed.

---

### weaponAlwaysRaised

**Reference**: `gamemode/config/sh_config.lua:160`

```lua
ix.config.Add("weaponAlwaysRaised", false, "Whether or not weapons are always raised.", nil, {
    category = "server"
})
```

**Type**: Boolean
**Default**: `false`
**Description**: When enabled, weapons are automatically raised without needing to use the raise key

**Usage**:
```lua
-- Check if weapons auto-raise
if ix.config.Get("weaponAlwaysRaised") then
    -- Weapons are always raised
end
```

---

### weaponRaiseTime

**Reference**: `gamemode/config/sh_config.lua:163`

```lua
ix.config.Add("weaponRaiseTime", 1, "The time it takes for a weapon to raise.", nil, {
    data = {min = 0.1, max = 60, decimals = 1},
    category = "server"
})
```

**Type**: Number (Float)
**Default**: `1`
**Range**: 0.1 - 60 seconds
**Description**: Time in seconds it takes for a weapon to be raised before it can be fired

**Usage**:
```lua
-- Get weapon raise time
local raiseTime = ix.config.Get("weaponRaiseTime", 1)
```

---

### allowBusiness

**Reference**: `gamemode/config/sh_config.lua:167`

```lua
ix.config.Add("allowBusiness", true, "Whether or not business is enabled.", nil, {
    category = "server"
})
```

**Type**: Boolean
**Default**: `true`
**Description**: Enables or disables the business/property ownership system

**Usage**:
```lua
-- Check if business system is enabled
if ix.config.Get("allowBusiness") then
    -- Business system is available
end
```

## Complete Example: Checking Server Config

```lua
-- Check server settings before performing actions
function PLUGIN:CanPlayerUseVoice(client)
    -- Check if voice is enabled server-wide
    if not ix.config.Get("allowVoice", false) then
        client:Notify("Voice chat is disabled on this server")
        return false
    end

    return true
end

-- Use weapon raise time in custom weapon
SWEP.RaiseTime = ix.config.Get("weaponRaiseTime", 1)

-- Check if business system is available
function PLUGIN:ShowBusinessMenu(client)
    if not ix.config.Get("allowBusiness", true) then
        client:Notify("Business system is disabled")
        return
    end

    -- Show business menu
end
```

## Best Practices

### ✅ DO

- Use `ix.config.Get()` to retrieve config values at runtime
- Provide default values as second parameter to `Get()`
- Check config values before using related systems
- Use config callbacks to respond to changes
- Let admins change values through F1 menu
- Define configs in schema or plugin for custom settings

### ❌ DON'T

- Don't hardcode server settings
- Don't create custom config files instead of using ix.config
- Don't modify config values directly in code (use admin menu)
- Don't bypass config system with global variables
- Don't forget to check if systems are enabled before using them

## Common Patterns

### Pattern 1: Conditional Feature Usage

```lua
-- Only enable feature if config allows it
function PLUGIN:PlayerCanUseFeature(client)
    if not ix.config.Get("allowFeature") then
        return false
    end

    -- Feature is enabled, perform action
    return true
end
```

### Pattern 2: Using Config with Callbacks

```lua
-- Schema config with callback
ix.config.Add("maxStamina", 100, "Maximum player stamina", function(oldValue, newValue)
    -- Update all players when config changes
    for _, client in player.Iterator() do
        client:SetNWInt("maxStamina", newValue)
    end
end, {
    category = "server",
    data = {min = 50, max = 500}
})
```

## Adding Custom Server Config

To add your own server config options in schemas or plugins:

```lua
-- In schema or plugin sh_plugin.lua
ix.config.Add("myServerOption", true, "Description of my option", function(oldValue, newValue)
    -- Optional callback when value changes
    if (SERVER) then
        -- React to change
    end
end, {
    category = "server",
    -- Optional constraints
    data = {min = 1, max = 100}
})
```

## Common Issues

### Issue: Config Changes Not Taking Effect

**Cause**: Config values are cached or not reloaded properly
**Fix**: Restart the server or ensure callbacks are implemented correctly

```lua
-- Use callbacks to apply changes immediately
ix.config.Add("myOption", 10, "My option", function(oldValue, newValue)
    -- Apply changes immediately when admin changes config
    ApplyNewValue(newValue)
end)
```

### Issue: Can't Find Config Value

**Cause**: Config option may not be registered yet
**Fix**: Always provide default value and check registration order

```lua
-- Always provide default
local value = ix.config.Get("myOption", defaultValue)

-- Ensure config is registered before using it
hook.Add("InitializedConfig", "MyPlugin", function()
    -- Now safe to use config values
    local value = ix.config.Get("myOption")
end)
```

## See Also

- [Configuration System](../systems/configuration.md) - Overview of config system
- [Character Configuration](characters.md) - Character-related settings
- [Chat Configuration](chat.md) - Chat system settings
- [Appearance Configuration](appearance.md) - Visual settings
- Source: `gamemode/config/sh_config.lua`
