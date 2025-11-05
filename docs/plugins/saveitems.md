# Save Items Plugin

> **Reference**: `plugins/saveitems.lua`

The Save Items plugin saves dropped item entities across server restarts. When an item is dropped in the world and the server restarts, the item respawns in the same location with the same state, preventing loss of dropped items and maintaining world persistence.

## ⚠️ Important: Use Built-in Item Persistence

**The plugin automatically saves all dropped items**. The framework provides:
- Automatic saving of all `ix_item` entities on server shutdown
- Automatic spawning on server start
- Position, angle, and physics state preservation
- Bag inventory restoration
- Optional deletion via hook

## Core Concepts

### What Gets Saved?

- **Item Entities**: All `ix_item` entities in the world
- **Position & Angles**: Exact placement preserved
- **Physics State**: Moveable/frozen state preserved
- **Bag Inventories**: Contents of dropped bags restored
- **Excludes Temporary**: Items marked `bTemporary` are not saved

## Developer Hooks

### ShouldDeleteSavedItems

**Reference**: `plugins/saveitems.lua:33-37`

```lua
-- Delete saved items instead of spawning them
function PLUGIN:ShouldDeleteSavedItems()
    -- Return true to delete saved items from database
    -- Return false/nil to spawn saved items

    -- Example: Delete on map change
    if game.GetMap() != PLUGIN.lastMap then
        return true
    end
end
```

### OnSavedItemLoaded

**Reference**: `plugins/saveitems.lua:7-11, 93`

```lua
-- Called after saved items are loaded
function PLUGIN:OnSavedItemLoaded(items)
    -- items: table of loaded item objects

    print("Loaded " .. #items .. " saved items")

    for _, item in ipairs(items) do
        print("Restored: " .. item.name)
    end
end
```

## Complete Examples

### Example 1: Delete Items on New Map

```lua
function PLUGIN:ShouldDeleteSavedItems()
    local lastMap = self:GetData("lastMap")

    if lastMap != game.GetMap() then
        self:SetData("lastMap", game.GetMap())
        return true  -- Delete old items
    end

    return false  -- Keep items
end
```

### Example 2: Custom Item Spawn Logic

```lua
function PLUGIN:OnSavedItemLoaded(items)
    for _, item in ipairs(items) do
        if item.uniqueID == "special_artifact" then
            -- Special handling for artifacts
            local ent = ents.GetByIndex(item.id)
            if IsValid(ent) then
                local effectData = EffectData()
                effectData:SetOrigin(ent:GetPos())
                util.Effect("ElectricSpark", effectData)
            end
        end
    end
end
```

### Example 3: Prevent Saving Specific Items

```lua
-- Mark items as temporary before server shutdown
function PLUGIN:SaveData()
    for _, ent in ipairs(ents.FindByClass("ix_item")) then
        local item = ix.item.instances[ent.ixItemID]

        if item and item.noDrop then
            ent.bTemporary = true  -- Won't be saved
        end
    end
end
```

## Best Practices

### ✅ DO

- Use ShouldDeleteSavedItems for cleanup
- Mark quest/temporary items with `bTemporary`
- Test with bags containing items
- Clean up old items periodically

### ❌ DON'T

- Don't rely on this for critical item storage (use inventories)
- Don't save thousands of items (performance)
- Don't forget to handle map changes

## See Also

- [Inventory System](../systems/inventory.md) - Item storage
- [Items System](../systems/items.md) - Item creation
- [Persistence Plugin](persistence.md) - Entity persistence
- Source: `plugins/saveitems.lua`
