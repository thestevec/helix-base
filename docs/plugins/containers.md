# Containers Plugin

> **Reference**: `plugins/containers/`

The Containers plugin transforms spawned props into interactive storage containers with persistent inventories. Allows players to create storage solutions by spawning registered container models, each with customizable inventory sizes and optional passwords.

## ⚠️ Important: Use Built-in Container System

**Always use `ix.container.Register()`** to define containers. The framework provides:
- Automatic prop-to-container conversion
- Persistent inventory storage
- Password protection support
- Money storage in containers
- Custom display names

## Registering Containers

**Reference**: `plugins/containers/sh_definitions.lua`

```lua
ix.container.Register(model, data)

-- Example:
ix.container.Register("models/props_c17/suitcase01a.mdl", {
    name = "Suitcase",
    description = "A small suitcase for storage.",
    width = 4,
    height = 3
})
```

## Configuration

- `containerSave` (default: true) - Save containers across restarts
- `containerOpenTime` (default: 0.7) - Time to open containers

## See Also

- [Inventory System](../systems/inventory.md) - Inventory management
- Source: `plugins/containers/`
