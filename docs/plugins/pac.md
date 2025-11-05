# PAC3 Integration Plugin

> **Reference**: `plugins/pac.lua`

The PAC3 Integration plugin integrates PAC3 (Player Appearance Customizer 3) with Helix's item system. Allows items to have PAC3 parts that attach when equipped, enabling custom models, accessories, and visual effects.

## ⚠️ Important: Requires PAC3

**This plugin requires PAC3 addon installed**. If PAC3 is not present, the plugin does nothing.

## Core Concepts

### What is PAC3 Integration?

- Items can define `pacData` containing PAC3 part information
- Parts automatically attach when item is equipped
- Parts automatically detach when item is unequipped
- Character-specific (parts reset on character change)

## Item Integration

### Defining PAC Item

```lua
ITEM.pacData = {
    -- PAC3 part data exported from PAC3 editor
}

-- Optional: Adjust parts per player
function ITEM:pacAdjust(parts, client)
    -- Modify parts table
    return parts
end
```

### Automatic Behavior

When item with `pacData` is equipped:
- `client:AddPart(uniqueID)` called
- PAC parts attach to player
- Parts visible to all players

When unequipped:
- `client:RemovePart(uniqueID)` called
- PAC parts detach

## Permissions

**Privilege**: `Helix - Manage PAC`
- Required to open PAC3 editor
- Required to wear custom parts
- Minimum access: superadmin

## Best Practices

### ✅ DO

- Export PAC3 parts properly
- Test parts on different models
- Use pacAdjust for model-specific adjustments

### ❌ DON'T

- Don't use excessive polygons
- Don't create laggy effects
- Don't bypass item system

## See Also

- [Items System](../systems/items.md) - Creating items
- [Outfits Item Type](../items/outfits.md) - PAC outfit items
- PAC3 Wiki: External documentation
- Source: `plugins/pac.lua`
