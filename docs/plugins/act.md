# Player Acts Plugin

> **Reference**: `plugins/act/`

The Player Acts plugin enables players to perform animations like sitting, leaning, saluting, and more. Acts are model-specific and can have multiple variants, allowing for rich character expression through animation sequences.

## ⚠️ Important: Use Built-in Acts System

**Always use `ix.act.Register()`** to create acts. The framework provides:
- Automatic command generation (e.g., `/ActSit`)
- Model-specific animation support
- Enter/exit sequences
- Timed or untimed animations
- Multiple variants per act

## Registering Acts

**Reference**: `plugins/act/sh_plugin.lua:23-59`

```lua
ix.act.Register(name, modelClass, data)

-- Example:
ix.act.Register("Sit", "player", {
    sequence = "sit_chair",
    untimed = true
})

-- Multiple variants:
ix.act.Register("Salute", "player", {
    sequence = {"salute1", "salute2", "salute3"}
})

-- With enter/exit:
ix.act.Register("Sit", "player", {
    sequence = "sit_chair",
    start = "sit_enter",
    finish = "sit_exit",
    untimed = true
})
```

## Act Structure

```lua
{
    sequence = "anim_name" or {"anim1", "anim2"},  -- Main animation(s)
    start = "enter_anim" or {"enter1", "enter2"},  -- Optional entry animation
    finish = "exit_anim" or {"exit1", "exit2"},    -- Optional exit animation
    untimed = true/false,  -- If true, animation loops until canceled
    weapon = false,        -- If false, prevents use while holding weapon
}
```

## Using Acts

### Commands

Acts automatically create commands:
- `/ActSit` - Perform sit animation
- `/ActSit 2` - Perform sit variant #2
- `/ActSalute` - Perform salute animation
- `/ActSalute 3` - Perform salute variant #3

### Exiting Acts

- Move (WASD keys)
- Jump
- Crouch
- Use `/CharFallOver` command

## Model Classes

Acts can be registered for:
- Specific models: `"models/player/model.mdl"`
- Model groups: `"player"`, `"combine"`, etc.
- Multiple models: `{"models/a.mdl", "models/b.mdl"}`

## Developer Functions

```lua
-- Remove an act
ix.act.Remove("ActName")

-- Check if player is in act
if client:GetNetVar("actEnterAngle") then
    -- Player is performing act
end
```

## Hooks

```lua
function PLUGIN:SetupActs()
    -- Register your custom acts here
    ix.act.Register("Dance", "player", {
        sequence = {"dance1", "dance2"}
    })
end
```

## Permissions

**Privilege**: `Helix - Player Acts`
Default: all users

## Pre-Registered Acts

The plugin includes many common acts:
- Sit/sit variations
- Lean against wall
- Salute
- Cheer
- And many more (see `sh_definitions.lua`)

## Best Practices

### ✅ DO

- Register acts in `SetupActs` hook
- Use untimed for looping animations
- Test animations with your player models
- Provide multiple variants for variety

### ❌ DON'T

- Don't use missing sequences
- Don't forget enter/exit sequences for sitting
- Don't make acts weapon-only without reason

## See Also

- [Animation System](../systems/animations.md) - Animation management
- Source: `plugins/act/`
