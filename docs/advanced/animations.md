# Animation Systems

> **Reference**: `gamemode/core/libs/sh_anims.lua`, `gamemode/core/libs/sh_animation.lua`

Helix provides two distinct animation systems: player model animation translation for supporting NPC models, and a tween animation system for creating smooth UI animations.

## ⚠️ Important: Use Built-in Animation Systems

**Always use Helix's built-in animation systems** rather than creating custom implementations. The framework provides:
- Automatic animation translation for NPC models (citizen, metrocop, overwatch, vortigaunt, etc.)
- Player sequence forcing with automatic cleanup
- Smooth tween animations for UI panels
- Integration with client animation preferences
- Network synchronization for player animations

## Core Concepts

### What are Helix Animations?

Helix provides two separate animation systems:

1. **Player Model Animations (ix.anim)**: Translates standard player movements to NPC model animations, allowing non-player models to be used as player characters without T-posing
2. **Tween Animations (CreateAnimation)**: Provides smooth interpolation for UI panel properties like position, size, color, and alpha

### Key Terms

- **Animation Class**: A set of animation translations for a specific model type (e.g., `citizen_male`, `metrocop`, `overwatch`)
- **Model Class**: The animation class assigned to a specific model path
- **Sequence**: A named animation that a player model can perform
- **Tween**: Smooth interpolation between two values over time
- **Easing**: The mathematical function controlling animation acceleration/deceleration

## Player Model Animations (ix.anim)

### Built-in Animation Classes

**Reference**: `gamemode/core/libs/sh_anims.lua:23-352`

Helix includes pre-configured animation classes:
- `citizen_male` - Male citizen models
- `citizen_female` - Female citizen models
- `metrocop` - Metro Police models
- `overwatch` - Combine soldier models
- `vortigaunt` - Vortigaunt models
- `player` - Standard player models
- `zombie` - Zombie models
- `fastZombie` - Fast zombie models

### ix.anim.SetModelClass

**Reference**: `gamemode/core/libs/sh_anims.lua:361`

```lua
ix.anim.SetModelClass(model, class)
```

Assigns an animation class to a specific model path. Use this when custom models T-pose.

**Complete Example**:
```lua
-- In your schema or plugin
function PLUGIN:InitializedConfig()
    -- Assign metrocop animations to a custom police model
    ix.anim.SetModelClass("models/custom/police_officer.mdl", "metrocop")

    -- Assign citizen_male animations to a custom civilian model
    ix.anim.SetModelClass("models/custom/businessman.mdl", "citizen_male")

    -- Assign vortigaunt animations to a custom alien model
    ix.anim.SetModelClass("models/custom/alien_worker.mdl", "vortigaunt")
end
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't modify animation bones manually
function PLUGIN:CalcMainActivity(client)
    client:SetPoseParameter("move_x", 1)  -- Don't bypass framework!
    client:ManipulateBonePosition(1, Vector(0, 0, 0))  -- Framework handles this!
end
```

### ix.anim.GetModelClass

**Reference**: `gamemode/core/libs/sh_anims.lua:376`

```lua
local class = ix.anim.GetModelClass(model)
```

Returns the animation class assigned to a model. Automatically detects `player`, `citizen_male`, and `citizen_female` models.

**Complete Example**:
```lua
function PLUGIN:SomeHook(client)
    local model = client:GetModel()
    local animClass = ix.anim.GetModelClass(model)

    if animClass == "metrocop" then
        -- This model uses metrocop animations
        print("Player is using metrocop animations")
    elseif animClass == "citizen_female" then
        -- This model uses female citizen animations
        print("Player is using female citizen animations")
    end
end
```

### Player:ForceSequence

**Reference**: `gamemode/core/libs/sh_anims.lua:423`

```lua
player:ForceSequence(sequence, callback, time, bNoFreeze)
```

Forces a player to perform a specific animation sequence. Prevents weapon firing and optionally freezes movement.

**Parameters**:
- `sequence` (string): Name of the animation sequence
- `callback` (function, optional): Called when animation completes
- `time` (number, optional): Duration override (defaults to sequence duration)
- `bNoFreeze` (boolean, optional): If true, player can move during animation

**Complete Example**:
```lua
-- In a command or interaction
function PLUGIN:PerformSitAnimation(client, entity)
    -- Make player sit in a chair with callback
    client:ForceSequence("sit", function()
        client:Notify("You stood up from the chair")
        client:SetPos(client:GetPos() + Vector(0, 0, 20))
    end, 5, false)  -- 5 seconds, frozen in place
end

-- Multi-stage animation sequence
function PLUGIN:PerformComplexAction(client)
    local startSequence = client:LookupSequence("gesture_becon")

    if startSequence > 0 then
        client:ForceSequence(startSequence, function()
            -- First animation done, start second
            client:ForceSequence("wave", function()
                -- Both animations complete
                client:Notify("Action completed!")
            end, 3)
        end)
    end
end

-- Animation without freezing player
function PLUGIN:PerformWaveWhileWalking(client)
    client:ForceSequence("wave", nil, nil, true)  -- Player can still move
end
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't use Entity:ResetSequence directly
client:ResetSequence("sit")  -- Won't network properly, no cleanup!

-- WRONG: Don't freeze player manually
client:SetMoveType(MOVETYPE_NONE)
client:ResetSequence("sit")  -- Framework handles this!
```

### Player:LeaveSequence

**Reference**: `gamemode/core/libs/sh_anims.lua:472`

```lua
player:LeaveSequence()
```

Stops a forced animation sequence and restores normal player control.

**Complete Example**:
```lua
-- Stop animation early based on condition
function PLUGIN:PlayerTakeDamage(client, attacker, inflictor, damage)
    if client:GetNetVar("forcedSequence") then
        -- Interrupt sitting animation when player takes damage
        client:LeaveSequence()
        client:Notify("You were forced to stand!")
    end
end

-- Stop all animations when player enters vehicle
function PLUGIN:CanPlayerEnterVehicle(client, vehicle)
    if client:GetNetVar("forcedSequence") then
        client:LeaveSequence()
    end

    return true
end
```

## Tween Animations (UI)

### Panel:CreateAnimation

**Reference**: `gamemode/core/libs/sh_animation.lua:66`

```lua
local animation = panel:CreateAnimation(length, data)
```

Creates a smooth tween animation for panel properties. Commonly used for fading, moving, and resizing UI elements.

**Parameters**:
- `length` (number): Animation duration in seconds
- `data` (table): Animation configuration
  - `target` (table): Properties to animate (e.g., `{Alpha = 255}`)
  - `easing` (string, optional): Easing function (default: "linear")
  - `OnComplete` (function, optional): Called when animation finishes
  - `Think` (function, optional): Called every frame during animation
  - `bAutoFire` (boolean, optional): Auto-start animation (default: true)
  - `bRemoveOnComplete` (boolean, optional): Remove animation when done (default: true)

**Complete Example**:
```lua
-- Fade in a panel
function PANEL:Init()
    self:SetAlpha(0)

    self:CreateAnimation(0.5, {
        target = {Alpha = 255},
        easing = "outQuint"
    })
end

-- Slide panel from left with callback
function PANEL:SlideIn()
    local startX = -self:GetWide()
    self:SetPos(startX, 0)

    self:CreateAnimation(0.3, {
        target = {x = 0},
        easing = "outCubic",
        OnComplete = function(animation, panel)
            panel:OnSlideComplete()
        end
    })
end

-- Fade out and remove panel
function PANEL:FadeOut()
    self:CreateAnimation(0.25, {
        target = {Alpha = 0},
        easing = "inQuint",
        OnComplete = function(anim, panel)
            panel:Remove()
        end
    })
end

-- Animate multiple properties
function PANEL:ExpandPanel()
    local currentWide, currentTall = self:GetSize()

    self:CreateAnimation(0.4, {
        target = {
            wide = currentWide * 1.5,
            tall = currentTall * 1.5,
            Alpha = 200
        },
        easing = "outElastic"
    })
end
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't animate manually in Think
function PANEL:Think()
    local alpha = self:GetAlpha()
    self:SetAlpha(math.Approach(alpha, 255, 5))  -- Don't do this!
end

-- WRONG: Don't use timer-based animations
timer.Create("MyPanelFade", 0.01, 100, function()
    if IsValid(panel) then
        panel:SetAlpha(panel:GetAlpha() + 2.55)  -- Use CreateAnimation!
    end
end)
```

### Animation Chaining

**Reference**: `gamemode/core/libs/sh_animation.lua:103`

```lua
animation:CreateAnimation(length, data)
```

Chain animations to run sequentially.

**Complete Example**:
```lua
-- Fade in, wait, then fade out
function PANEL:FlashMessage()
    self:SetAlpha(0)

    -- Fade in
    local fadeIn = self:CreateAnimation(0.3, {
        target = {Alpha = 255},
        easing = "outQuint"
    })

    -- Chain: wait (no change), then fade out
    fadeIn:CreateAnimation(1.5, {
        target = {Alpha = 255}  -- No change, just delay
    }):CreateAnimation(0.3, {
        target = {Alpha = 0},
        easing = "inQuint",
        OnComplete = function(anim, panel)
            panel:Remove()
        end
    })
end

-- Complex multi-stage animation
function PANEL:ShowWithEffect()
    self:SetAlpha(0)
    self:SetSize(0, 0)

    -- Stage 1: Fade in
    self:CreateAnimation(0.2, {
        target = {Alpha = 255}
    }):CreateAnimation(0.3, {
        -- Stage 2: Expand width
        target = {wide = 400}
    }):CreateAnimation(0.3, {
        -- Stage 3: Expand height
        target = {tall = 300}
    })
end
```

## Common Patterns

### Pattern 1: Sit in Chair Animation

```lua
-- Make player sit and attach to chair
function PLUGIN:PlayerUseDoor(client, entity)
    if entity.isChair then
        local sitPos = entity:GetPos()
        client:SetPos(sitPos)

        client:ForceSequence("sit", function()
            -- Player stood up
            client:SetPos(sitPos + Vector(0, 0, 20))
        end, 0, false)  -- 0 = use full sequence duration

        return false  -- Prevent door use
    end
end
```

### Pattern 2: Notification Panel Animation

```lua
-- Create notification that slides in and fades out
function SCHEMA:CreateNotification(text)
    local panel = vgui.Create("DPanel")
    panel:SetSize(300, 50)
    panel:SetPos(-300, 100)  -- Start off-screen

    -- Slide in
    panel:CreateAnimation(0.3, {
        target = {x = 20},
        easing = "outBack"
    }):CreateAnimation(3, {
        -- Wait for 3 seconds
        target = {}
    }):CreateAnimation(0.3, {
        -- Fade and slide out
        target = {
            Alpha = 0,
            x = -300
        },
        OnComplete = function(anim, pnl)
            pnl:Remove()
        end
    })
end
```

### Pattern 3: Custom Model Animation Setup

```lua
-- Register custom models with animation classes
function SCHEMA:InitializedConfig()
    -- Custom rebel models use citizen animations
    ix.anim.SetModelClass("models/custom/rebel_01.mdl", "citizen_male")
    ix.anim.SetModelClass("models/custom/rebel_02.mdl", "citizen_male")

    -- Custom female rebels
    ix.anim.SetModelClass("models/custom/rebel_female_01.mdl", "citizen_female")

    -- Custom combine models
    ix.anim.SetModelClass("models/custom/elite_soldier.mdl", "overwatch")
end
```

### Pattern 4: Interactive Animation System

```lua
-- Animation with player interaction
PLUGIN.animations = {
    ["sit"] = {sequence = "sit", freeze = true},
    ["wave"] = {sequence = "wave", freeze = false},
    ["salute"] = {sequence = "salute", freeze = false}
}

function PLUGIN:PlayerBindPress(client, bind, pressed)
    if bind == "+use" and pressed then
        local trace = client:GetEyeTrace()
        local entity = trace.Entity

        if IsValid(entity) and entity.animationTrigger then
            local animData = self.animations[entity.animationTrigger]

            if animData then
                client:ForceSequence(animData.sequence, function()
                    client:Notify("Animation complete!")
                end, nil, not animData.freeze)

                return true
            end
        end
    end
end
```

## Best Practices

### ✅ DO

- Use `ix.anim.SetModelClass()` for custom NPC models to prevent T-posing
- Check if sequence exists with `player:LookupSequence()` before forcing
- Use `CreateAnimation()` for all UI animations instead of manual tweening
- Respect client animation settings (framework handles this automatically)
- Chain animations using `animation:CreateAnimation()` for sequential effects
- Use easing functions for smooth, professional animations
- Clean up animations with callbacks or `bRemoveOnComplete`

### ❌ DON'T

- Don't modify player bones or animations directly—use `ForceSequence()`
- Don't create timer-based animations—use `CreateAnimation()`
- Don't forget to call `LeaveSequence()` when interrupting animations
- Don't animate every frame in `Think()`—use tween system
- Don't hardcode animation classes for all models—use `GetModelClass()`
- Don't bypass animation preferences with `bIgnoreConfig = true` unless necessary
- Don't forget to handle animation callbacks for cleanup

## Common Issues

### Issue: Model T-posing

**Cause**: Model doesn't have a registered animation class
**Fix**: Assign appropriate animation class

```lua
-- Set animation class for custom model
ix.anim.SetModelClass("models/your/custom/model.mdl", "citizen_male")

-- Or in plugin initialization
function PLUGIN:InitializedConfig()
    ix.anim.SetModelClass("models/mymod/npc.mdl", "metrocop")
end
```

### Issue: Animation Won't Stop

**Cause**: Forgot to call `LeaveSequence()` or provide duration
**Fix**: Ensure proper cleanup

```lua
-- Provide explicit duration
client:ForceSequence("sit", callback, 5)  -- Will auto-stop after 5 seconds

-- Or manually stop when needed
if client:GetNetVar("forcedSequence") then
    client:LeaveSequence()
end
```

### Issue: UI Animation Doesn't Start

**Cause**: `bAutoFire = false` without manual `Fire()` call
**Fix**: Either enable auto-fire or manually start

```lua
-- Auto-start (default)
panel:CreateAnimation(0.5, {
    target = {Alpha = 255}
})

-- Or manual start
local anim = panel:CreateAnimation(0.5, {
    target = {Alpha = 255},
    bAutoFire = false
})
anim:Fire()
```

### Issue: Player Stuck After Animation

**Cause**: `ForceSequence()` doesn't restore movement
**Fix**: Use `LeaveSequence()` or ensure callback executes

```lua
-- Always restore control
client:ForceSequence("sit", function()
    client:LeaveSequence()  -- Explicitly restore control
end, 5)

-- Or let framework handle it automatically
client:ForceSequence("sit", nil, 5)  -- Auto-restores after 5 seconds
```

## See Also

- [Character System](../systems/character.md) - Character model management
- [Plugin System](../plugins/plugin-system.md) - Creating custom animations in plugins
- [UI Development](../guides/ui-development.md) - Creating animated interfaces
- Source: `gamemode/core/libs/sh_anims.lua`
- Source: `gamemode/core/libs/sh_animation.lua`
- Example Usage: `plugins/act/sh_plugin.lua`
