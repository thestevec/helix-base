# HUD System (ix.bar)

> **Reference**: `gamemode/core/libs/cl_bar.lua`, `gamemode/core/derma/cl_bar.lua`

The HUD bar system provides status indicators displayed on-screen, such as health, armor, and custom attributes. It includes both persistent info bars and temporary action progress bars.

## ⚠️ Important: Use Built-in ix.bar Functions

**Always use Helix's built-in `ix.bar` system** rather than creating custom HUD elements. The framework provides:
- Automatic positioning and organization
- Built-in animations and smooth transitions
- Priority-based sorting
- Auto-hide functionality with lifetime management
- User preference integration (`alwaysShowBars` option)
- Consistent styling through derma skin system

## Core Concepts

### What is the HUD Bar System?

The HUD bar system consists of two main components:

1. **Info Bars** - Persistent status bars (health, armor, attributes) that appear in the top-left corner
2. **Action Bar** - Temporary progress bar that appears in the center-bottom of the screen during timed actions

### Key Terms

- **Bar** - A status indicator showing a value from 0-1 (0% to 100%)
- **Priority** - Determines display order (lower priority = displayed higher on screen)
- **Lifetime** - Auto-hide timer; bars hide after 5 seconds of inactivity unless marked visible
- **Action Bar** - Center-screen progress display for timed player actions
- **Delta** - Smoothly interpolated current value for animation

## Using Info Bars

### ix.bar.Add()

**Reference**: `gamemode/core/libs/cl_bar.lua:33`

Registers a new status bar to display in the HUD.

```lua
ix.bar.Add(getValue, color, priority, identifier)
```

**Parameters**:
- `getValue` (function) - Function that returns current value (0-1) and optional text
- `color` (Color, optional) - Bar color (random if not specified)
- `priority` (number, optional) - Display order (lower = higher on screen)
- `identifier` (string, optional) - Unique ID for later removal

**Returns**: Priority value assigned to the bar

```lua
-- Basic health bar (already included by default)
ix.bar.Add(function()
    local client = LocalPlayer()
    return math.max(client:Health() / client:GetMaxHealth(), 0)
end, Color(200, 50, 40), nil, "health")

-- Custom attribute bar with text
ix.bar.Add(function()
    local client = LocalPlayer()
    local character = client:GetCharacter()

    if character then
        local stamina = character:GetAttribute("stm", 0)
        local maxStamina = character:GetAttribute("stm", 0)

        return stamina / 100, string.format("Stamina: %d%%", stamina)
    end

    return false  -- Returning false hides the bar
end, Color(100, 200, 100), 3, "stamina")
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't draw custom HUD elements manually
hook.Add("HUDPaint", "MyCustomHUD", function()
    -- Manual drawing won't integrate with framework!
    surface.SetDrawColor(255, 0, 0)
    surface.DrawRect(10, 10, 200, 20)
end)

-- Use ix.bar.Add() instead
```

### ix.bar.Get()

**Reference**: `gamemode/core/libs/cl_bar.lua:13`

Retrieves a registered bar by its identifier.

```lua
local bar = ix.bar.Get(identifier)
```

```lua
-- Check if a bar exists
local healthBar = ix.bar.Get("health")
if healthBar then
    print("Health bar priority:", healthBar.priority)
end
```

### ix.bar.Remove()

**Reference**: `gamemode/core/libs/cl_bar.lua:21`

Removes a bar from the HUD by its identifier.

```lua
ix.bar.Remove(identifier)
```

```lua
-- Remove a custom bar
ix.bar.Remove("stamina")

-- Add it back later if needed
ix.bar.Add(getValueFunc, color, priority, "stamina")
```

## Using Action Bars

### Player:SetAction()

**Reference**: `gamemode/core/meta/sh_player.lua:269`

Displays a temporary progress bar for timed actions (SERVER-SIDE).

```lua
player:SetAction(text, time, callback, startTime, finishTime)
```

**Parameters**:
- `text` (string, optional) - Display text (use "@phrase" for localization)
- `time` (number, optional) - Duration in seconds
- `callback` (function, optional) - Function called on completion
- `startTime` (number, optional) - Custom start time (CurTime())
- `finishTime` (number, optional) - Custom finish time

```lua
-- Basic action with callback
player:SetAction("@searching", 5, function()
    if IsValid(player) and player:GetCharacter() then
        player:Notify("Search complete!")
    end
end)

-- Action without callback
player:SetAction("@respawning", 10)

-- Cancel an action
player:SetAction()  -- No parameters = cancel
```

### Player:DoStaredAction()

**Reference**: `gamemode/core/meta/sh_player.lua:126`

Creates an action that requires the player to look at an entity.

```lua
player:DoStaredAction(entity, callback, time, onCancel, distance)
```

**Parameters**:
- `entity` (Entity) - Entity player must look at
- `callback` (function) - Function called on success
- `time` (number) - Duration in seconds
- `onCancel` (function, optional) - Function called if cancelled
- `distance` (number, optional) - Maximum look-away distance (default: 96)

```lua
-- Require player to look at door while unlocking
player:DoStaredAction(door, function()
    door:Fire("Unlock")
    player:Notify("Door unlocked!")
end, 3, function()
    player:Notify("You looked away!")
end)

-- Combined with SetAction for visual feedback
player:SetAction("@unlocking", 3)
player:DoStaredAction(door, function()
    door:Fire("Unlock")
end, 3, function()
    player:SetAction()  -- Cancel action bar
end)
```

## Complete Example: Custom Attribute Bar

```lua
-- CLIENT-SIDE: Add stamina bar
if CLIENT then
    hook.Add("InitializedPlugins", "MyPlugin.AddStaminaBar", function()
        ix.bar.Add(function()
            local client = LocalPlayer()
            local character = client:GetCharacter()

            if not character then
                return false  -- Hide when no character
            end

            local stamina = character:GetAttribute("stm", 0)
            local maxStamina = 100

            -- Return value (0-1) and optional text
            return stamina / maxStamina, L("stamina") .. ": " .. math.Round(stamina)
        end, Color(100, 200, 100), 3, "stamina")
    end)
end

-- SERVER-SIDE: Use action bar during stamina-consuming action
function PLUGIN:PlayerSprint(client)
    if not client.ixStaminaAction then
        client:SetAction("@running", 999)  -- Long duration
        client.ixStaminaAction = true
    end
end

function PLUGIN:FinishSprinting(client)
    client:SetAction()  -- Cancel action bar
    client.ixStaminaAction = false
end
```

## Complete Example: Container Search

```lua
-- SERVER-SIDE: Timed container search
function OpenContainer(player, container)
    -- Show action bar for search duration
    player:SetAction("@searching", 5, function()
        if IsValid(player) and IsValid(container) and player:GetCharacter() then
            -- Open container inventory on completion
            local inventory = container:GetInventory()
            if inventory then
                inventory:Sync(player)
                player:Notify("Container opened!")
            end
        end
    end)

    -- Require player to look at container
    player:DoStaredAction(container, function()
        -- Success handled by SetAction callback
    end, 5, function()
        -- Cancelled - reset action bar
        player:SetAction()
        player:Notify("Search interrupted!")
    end)
end
```

## Bar Visibility Control

### Hooks

**Reference**: `gamemode/core/derma/cl_bar.lua:109-127`

```lua
-- Hide all bars (e.g., for cinematic mode)
hook.Add("ShouldHideBars", "MyPlugin.HideBars", function()
    return LocalPlayer():GetNoDraw()  -- Hide when player invisible
end)

-- Control individual bar visibility
hook.Add("ShouldBarDraw", "MyPlugin.ControlBars", function(barInfo)
    if barInfo.identifier == "health" then
        return LocalPlayer():Health() < LocalPlayer():GetMaxHealth()
    end
end)
```

### User Option

Players can toggle bar visibility:

```lua
-- Bars always visible (no auto-hide)
ix.option.Get("alwaysShowBars", false)

-- Check in code
if ix.option.Get("alwaysShowBars") then
    -- Bars won't auto-hide after lifetime expires
end
```

## Panel Components

### ixInfoBarManager

**Reference**: `gamemode/core/derma/cl_bar.lua:4-141`

The manager panel that contains and organizes all info bars. Automatically created at `ix.gui.bars`.

### ixInfoBar

**Reference**: `gamemode/core/derma/cl_bar.lua:144-192`

Individual bar panel with smooth value interpolation.

```lua
-- Access bar panels (usually not needed)
for _, barPanel in ipairs(ix.gui.bars:GetAll()) do
    print("Bar ID:", barPanel:GetID())
    print("Bar color:", barPanel:GetColor())
    print("Bar value:", barPanel:GetValue())
end
```

## Best Practices

### ✅ DO

- Use `ix.bar.Add()` for status displays
- Return `false` from `getValue` to hide bars dynamically
- Use unique identifiers for bars you may need to remove
- Call `player:SetAction()` with no args to cancel actions
- Use `@localizationPhrase` for action bar text
- Combine `SetAction()` and `DoStaredAction()` for interactive tasks
- Set appropriate priorities (health/armor use 1-2)
- Return both value and text from `getValue` functions

### ❌ DON'T

- Don't draw custom HUD elements with `HUDPaint` hook
- Don't create bars without checking if character exists
- Don't forget to cancel action bars on failure
- Don't use `SetAction()` client-side (server only)
- Don't hardcode bar text (use localization)
- Don't create duplicate bars (use identifiers)
- Don't forget callbacks can run after player/entity invalid

## Common Patterns

### Pattern 1: Conditional Bar Display

```lua
-- Only show bar when relevant
ix.bar.Add(function()
    local client = LocalPlayer()
    local character = client:GetCharacter()

    if not character then
        return false  -- Hide when no character
    end

    local value = character:GetData("customValue", 0)

    if value <= 0 then
        return false  -- Hide when at minimum
    end

    return math.Clamp(value / 100, 0, 1), "Custom: " .. value
end, Color(150, 100, 200), 5, "customBar")
```

### Pattern 2: Interruptible Action

```lua
-- SERVER-SIDE: Action that can be interrupted
function StartCrafting(player, duration)
    player:SetAction("@crafting", duration, function()
        if IsValid(player) and player:Alive() then
            -- Complete crafting
            CreateCraftedItem(player)
        end
    end)

    -- Cancel if player moves
    local startPos = player:GetPos()
    local hookName = "CraftingCheck_" .. player:SteamID()

    hook.Add("Tick", hookName, function()
        if not IsValid(player) then
            hook.Remove("Tick", hookName)
            return
        end

        if player:GetPos():Distance(startPos) > 50 then
            player:SetAction()  -- Cancel
            player:Notify("Crafting interrupted!")
            hook.Remove("Tick", hookName)
        end
    end)
end
```

### Pattern 3: Multiple Attribute Bars

```lua
-- CLIENT-SIDE: Add bars for all character attributes
hook.Add("InitializedPlugins", "AddAttributeBars", function()
    local attributes = {
        {id = "stm", name = "Stamina", color = Color(100, 200, 100), priority = 3},
        {id = "str", name = "Strength", color = Color(200, 100, 100), priority = 4},
        {id = "spd", name = "Speed", color = Color(100, 150, 200), priority = 5}
    }

    for _, attr in ipairs(attributes) do
        ix.bar.Add(function()
            local client = LocalPlayer()
            local character = client:GetCharacter()

            if not character then
                return false
            end

            local value = character:GetAttribute(attr.id, 0)
            return value / 100, attr.name .. ": " .. math.Round(value)
        end, attr.color, attr.priority, "attr_" .. attr.id)
    end
end)
```

## Common Issues

### Action Bar Doesn't Appear

**Cause**: Called on client-side or player has no character
**Fix**: Always use `SetAction()` server-side

```lua
-- WRONG (client-side)
if CLIENT then
    LocalPlayer():SetAction("@searching", 5)
end

-- CORRECT (server-side)
if SERVER then
    player:SetAction("@searching", 5)
end
```

### Bar Shows Wrong Value

**Cause**: getValue function not clamping or checking validity
**Fix**: Always clamp values to 0-1 range and validate

```lua
-- WRONG
ix.bar.Add(function()
    return LocalPlayer():Health() / 100  -- Max health might not be 100!
end)

-- CORRECT
ix.bar.Add(function()
    local client = LocalPlayer()
    return math.Clamp(client:Health() / client:GetMaxHealth(), 0, 1)
end)
```

### Action Callback Runs After Death

**Cause**: Callback doesn't validate player state
**Fix**: Always check validity in callbacks

```lua
-- WRONG
player:SetAction("@respawning", 10, function()
    player:Spawn()  -- Player might be invalid!
end)

-- CORRECT
player:SetAction("@respawning", 10, function()
    if IsValid(player) and player:GetCharacter() then
        player:Spawn()
    end
end)
```

### DoStaredAction Cancels Immediately

**Cause**: Entity is invalid or player not looking at it
**Fix**: Validate entity and ensure player is facing it

```lua
-- Check before starting
if not IsValid(entity) or player:GetEyeTrace().Entity != entity then
    player:Notify("You must look at the target!")
    return
end

player:DoStaredAction(entity, callback, time)
```

## See Also

- [Derma Overview](derma-overview.md) - UI panel system
- [Configuration System](../systems/configuration.md) - `alwaysShowBars` option
- [Attributes System](../systems/attributes.md) - Character attributes
- [Player Meta](../api/player.md) - Player methods
- Source: `gamemode/core/libs/cl_bar.lua`
- Source: `gamemode/core/derma/cl_bar.lua`
- Source: `gamemode/core/meta/sh_player.lua` (lines 118-170, 269-300)
