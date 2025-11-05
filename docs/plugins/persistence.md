# Persistence Plugin

> **Reference**: `plugins/persistence.lua`

The Persistence plugin allows administrators to mark props and entities as persistent, saving them across server restarts. Perfect for permanent decorations, furniture, and environmental objects.

## ⚠️ Important: Use Built-in Persistence

**Always use the physgun context menu** to make props persistent. The framework provides:
- Right-click properties menu integration
- Automatic saving of all entity properties
- Prevents physgun pickup of persistent props
- Logging of persistence changes

## Using Persistence

### Making Props Persistent

1. Spawn a prop
2. Right-click with physgun
3. Select "Make Persistent" from context menu
4. Prop will respawn on server restart

### Stopping Persistence

1. Right-click persistent prop with physgun
2. Select "Stop Persisting"
3. Prop will not respawn after restart

## Saved Properties

- Position and angles
- Model, skin, material
- Color
- Body groups
- Sub-materials
- Physics state (frozen/moveable)
- Collision group

## Preventing Persistence

```lua
-- In entity definition
ENT.bNoPersist = true  -- Cannot be made persistent
```

## Best Practices

### ✅ DO

- Use for permanent decorations
- Freeze persistent props
- Keep count reasonable (< 100 props)

### ❌ DON'T

- Don't persist temporary objects
- Don't persist vehicles or NPCs
- Don't create thousands of persistent props

## See Also

- [Save Items Plugin](saveitems.md) - Item persistence
- Source: `plugins/persistence.lua`
