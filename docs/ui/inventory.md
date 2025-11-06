# Inventory UI

> **Reference**: `gamemode/core/derma/cl_inventory.lua`

The inventory UI provides a drag-and-drop grid-based interface for managing character items. It's displayed in the main menu's default tab and handles item interactions, transfers, and combinations.

## ⚠️ Important: Use Built-in Inventory Panels

**Always use Helix's inventory panels** rather than creating custom inventory UIs. The framework provides:
- Automatic grid-based layout with proper collision detection
- Drag-and-drop functionality with visual feedback
- Right-click context menus for item actions
- Network synchronization with server
- Icon rendering with custom camera angles
- Item combination detection
- Transfer validation between inventories

## Core Concepts

### What is the Inventory UI?

The inventory UI consists of two main panel types:

1. **ixInventory** - The container panel with grid slots
2. **ixItemIcon** - Individual item icons that can be dragged

### Key Terms

- **Grid Position** - Item location in inventory (gridX, gridY)
- **Icon Size** - Size of each grid cell in pixels (default: 64)
- **Item Width/Height** - How many grid cells an item occupies
- **Drag-and-Drop** - Moving items between slots/inventories
- **Inventory Action** - Network message for item operations (move, drop, use, etc.)
- **Receiver** - Panel that accepts dragged items

## Inventory Panel (ixInventory)

### Creating an Inventory Panel

**Reference**: `gamemode/core/derma/cl_inventory.lua:282`

```lua
-- Create inventory display
local invPanel = vgui.Create("ixInventory")
invPanel:SetSize(400, 300)
invPanel:Center()
invPanel:MakePopup()

-- Set inventory to display
invPanel:SetInventory(inventoryObject)
```

### ixInventory:SetInventory()

**Reference**: `gamemode/core/derma/cl_inventory.lua` (methods throughout)

Associates inventory data with the panel.

```lua
-- Set which inventory to display
local character = LocalPlayer():GetCharacter()
local inventory = character:GetInventory()

invPanel:SetInventory(inventory)
```

### ixInventory:SetIconSize()

**Reference**: `gamemode/core/derma/cl_inventory.lua:279`

Sets the size of each grid cell.

```lua
-- Default size
invPanel:SetIconSize(64)

-- Smaller cells for compact display
invPanel:SetIconSize(48)

-- Larger cells for detailed view
invPanel:SetIconSize(80)
```

## Item Icon Panel (ixItemIcon)

### Item Icon Properties

**Reference**: `gamemode/core/derma/cl_inventory.lua:36-43`

Each item icon stores:
- `itemTable` - Reference to item data
- `inventoryID` - Which inventory contains the item
- `gridX`, `gridY` - Grid position
- `gridW`, `gridH` - Size in grid cells
- `slots` - Array of grid slots occupied

### Right-Click Context Menu

**Reference**: `gamemode/core/derma/cl_inventory.lua:67-181`

Right-clicking an item opens a context menu with available actions:

```lua
-- Context menu is created automatically from item functions
-- Each item function becomes a menu option

-- Example item with functions:
ITEM.functions.use = {
    name = "Use",
    icon = "icon16/accept.png",
    OnRun = function(item)
        -- Use the item
    end,
    OnCanRun = function(item)
        return true  -- Show this option
    end
}
```

### Drag-and-Drop Operations

**Reference**: `gamemode/core/derma/cl_inventory.lua:183-216`

Items can be dragged to:
- Move within same inventory
- Transfer between inventories
- Drop into world
- Combine with other items

```lua
-- Automatic drag-and-drop handling
-- OnDrop is called when item is released
function PANEL:OnDrop(bDragging, inventoryPanel, inventory, gridX, gridY)
    -- Validates and executes the drop operation
end
```

## Complete Example: Custom Storage Container

```lua
-- CLIENT-SIDE: Display storage container inventory
function OpenStorageInventory(containerEntity)
    -- Get container's inventory
    local inventory = containerEntity:GetInventory()

    if not inventory then
        return
    end

    -- Create container window
    local frame = vgui.Create("DFrame")
    frame:SetSize(500, 600)
    frame:Center()
    frame:SetTitle("Storage Container")
    frame:MakePopup()

    -- Character inventory at top
    local charInv = frame:Add("ixInventory")
    charInv:Dock(TOP)
    charInv:DockMargin(5, 5, 5, 5)
    charInv:SetIconSize(64)

    local character = LocalPlayer():GetCharacter()
    charInv:SetInventory(character:GetInventory())
    charInv:SetTitle("Your Inventory")

    -- Storage inventory at bottom
    local storageInv = frame:Add("ixInventory")
    storageInv:Dock(FILL)
    storageInv:DockMargin(5, 5, 5, 5)
    storageInv:SetIconSize(64)

    storageInv:SetInventory(inventory)
    storageInv:SetTitle("Storage")

    -- Close when container is removed
    frame.Think = function(self)
        if not IsValid(containerEntity) then
            self:Remove()
        end
    end

    return frame
end
```

## Complete Example: Custom Item Action

```lua
-- Add custom action to item context menu
hook.Add("CreateItemInteractionMenu", "CustomItemAction", function(panel, menu, itemTable)
    -- Add custom option
    menu:AddOption("Inspect", function()
        -- Open inspection UI
        OpenItemInspection(itemTable)
    end):SetImage("icon16/magnifier.png")

    -- Add option with sub-menu
    local subMenu = menu:AddSubMenu("Advanced")
    subMenu:AddOption("Split Stack", function()
        -- Split item stack
        OpenSplitDialog(itemTable)
    end)
    subMenu:AddOption("Mark Favorite", function()
        itemTable:SetData("favorite", true)
    end)

    -- Return true to completely override default menu
    -- Return nothing to add to default menu
end)
```

## Complete Example: Inventory Observer

```lua
-- CLIENT-SIDE: Monitor inventory changes
local function WatchInventory(inventory)
    local oldItems = {}

    -- Track existing items
    for _, item in pairs(inventory:GetItems()) do
        oldItems[item:GetID()] = true
    end

    -- Check for changes
    timer.Create("InventoryWatcher_" .. inventory:GetID(), 0.5, 0, function()
        if not inventory then
            timer.Remove("InventoryWatcher_" .. inventory:GetID())
            return
        end

        local currentItems = inventory:GetItems()

        -- Check for new items
        for _, item in pairs(currentItems) do
            if not oldItems[item:GetID()] then
                LocalPlayer():Notify("New item: " .. item:GetName())
                oldItems[item:GetID()] = true
            end
        end

        -- Check for removed items
        for id in pairs(oldItems) do
            if not currentItems[id] then
                oldItems[id] = nil
            end
        end
    end)
end

-- Start watching character inventory
hook.Add("CharacterLoaded", "WatchInventory", function(character)
    local inventory = character:GetInventory()
    if inventory then
        WatchInventory(inventory)
    end
end)
```

## Item Rendering

### Custom Icon Rendering

**Reference**: `gamemode/core/derma/cl_inventory.lua:8-25`

Items can specify custom icon camera angles:

```lua
-- In item definition
ITEM.iconCam = {
    pos = Vector(0, 0, 200),
    ang = Angle(90, 0, 0),
    fov = 45
}

-- Force re-render on next display
ITEM.forceRender = true
```

### External Rendering (exRender)

**Reference**: `gamemode/core/derma/cl_inventory.lua:666-683`

Items can use pre-rendered icons:

```lua
-- Use custom rendered icon
ITEM.exRender = true

-- Icon will be fetched via ikon system
-- ikon:GetIcon(itemTable.uniqueID)
```

## Inventory Actions

### Sending Actions to Server

**Reference**: `gamemode/core/derma/cl_inventory.lua:27-34`

```lua
-- Network format for item actions
local function InventoryAction(action, itemID, invID, data)
    net.Start("ixInventoryAction")
        net.WriteString(action)
        net.WriteUInt(itemID, 32)
        net.WriteUInt(invID, 32)
        net.WriteTable(data or {})
    net.SendToServer()
end

-- Common actions:
-- "use" - Use the item
-- "drop" - Drop into world
-- "transfer" - Move to another inventory
-- "combine" - Combine with another item
```

## Hooks

### CreateItemInteractionMenu

**Reference**: `gamemode/core/derma/cl_inventory.lua:75`

Modify or override item right-click menu.

```lua
hook.Add("CreateItemInteractionMenu", "MyPlugin", function(panel, menu, itemTable)
    -- panel: The ixItemIcon panel
    -- menu: DermaMenu to modify
    -- itemTable: Item data

    -- Add custom option
    menu:AddOption("Custom Action", function()
        RunItemAction(itemTable)
    end)

    -- Return true to completely replace menu
    -- Return nothing to add to existing menu
end)
```

### CanPlayerViewInventory

**Reference**: `gamemode/core/derma/cl_inventory.lua:743`

Control whether inventory tab appears in menu.

```lua
hook.Add("CanPlayerViewInventory", "MyPlugin", function()
    -- Return false to hide inventory tab
    if LocalPlayer():GetNoDraw() then
        return false
    end
end)
```

## Best Practices

### ✅ DO

- Use `vgui.Create("ixInventory")` for inventory displays
- Call `SetInventory()` to associate data with panel
- Use `SetIconSize()` for different display scales
- Let framework handle drag-and-drop validation
- Check `OnCanRun` in item functions to show/hide options
- Use `CreateItemInteractionMenu` hook to add custom actions
- Validate inventory exists before displaying
- Remove panels when associated entity/inventory invalid

### ❌ DON'T

- Don't create custom inventory grids (use ixInventory)
- Don't manually network item operations (use InventoryAction)
- Don't forget to check if inventory still valid
- Don't bypass item function callbacks
- Don't create item icons manually (let ixInventory do it)
- Don't forget `OnCanRun` checks in custom actions
- Don't assume items are in expected positions (always check)
- Don't forget to handle inventory access permissions

## Common Patterns

### Pattern 1: Split Inventory Display

```lua
-- Show two inventories side-by-side
function CreateSplitInventoryView(inv1, inv2, title1, title2)
    local frame = vgui.Create("DFrame")
    frame:SetSize(900, 500)
    frame:Center()
    frame:SetTitle("Transfer Items")
    frame:MakePopup()

    -- Left inventory
    local leftInv = frame:Add("ixInventory")
    leftInv:SetPos(10, 30)
    leftInv:SetSize(430, 460)
    leftInv:SetInventory(inv1)
    leftInv:SetTitle(title1)

    -- Right inventory
    local rightInv = frame:Add("ixInventory")
    rightInv:SetPos(460, 30)
    rightInv:SetSize(430, 460)
    rightInv:SetInventory(inv2)
    rightInv:SetTitle(title2)

    return frame
end
```

### Pattern 2: Filtered Item Display

```lua
-- Show only specific item types
hook.Add("CreateMenuButtons", "FilteredInventory", function(tabs)
    tabs["weapons"] = {
        Create = function(info, container)
            local invPanel = container:Add("ixInventory")
            invPanel:Dock(FILL)

            local inventory = LocalPlayer():GetCharacter():GetInventory()
            invPanel:SetInventory(inventory)

            -- Hide non-weapon items
            invPanel.Think = function(self)
                for _, panel in pairs(self.panels) do
                    if IsValid(panel) then
                        local item = panel:GetItemTable()
                        if item and item.base != "base_weapons" then
                            panel:SetVisible(false)
                        else
                            panel:SetVisible(true)
                        end
                    end
                end
            end
        end
    }
end)
```

### Pattern 3: Custom Item Tooltip

```lua
-- Show detailed tooltip on hover
hook.Add("PostRenderVGUI", "ItemTooltips", function()
    local panel = vgui.GetHoveredPanel()

    if IsValid(panel) and panel:GetClassName() == "ixItemIcon" then
        local item = panel:GetItemTable()

        if item then
            local x, y = input.GetCursorPos()

            -- Draw tooltip
            draw.SimpleText(item:GetName(), "ixMediumFont", x + 20, y, color_white)
            draw.SimpleText(item:GetDescription(), "ixSmallFont", x + 20, y + 20, Color(200, 200, 200))

            if item.price then
                draw.SimpleText("Value: " .. ix.currency.Get(item.price), "ixSmallFont", x + 20, y + 40, Color(100, 255, 100))
            end
        end
    end
end)
```

## Common Issues

### Items Not Appearing

**Cause**: Inventory not synced or panel not set up
**Fix**: Ensure inventory is synced and SetInventory called

```lua
-- WRONG
local invPanel = vgui.Create("ixInventory")
-- No inventory set!

-- CORRECT
local invPanel = vgui.Create("ixInventory")
local inventory = character:GetInventory()

if inventory then
    invPanel:SetInventory(inventory)
end
```

### Drag-and-Drop Not Working

**Cause**: Panels not registered as receivers or wrong receiver name
**Fix**: Use correct receiver name constant

```lua
-- Receiver name must match
local RECEIVER_NAME = "ixInventoryItem"

-- Already handled by ixInventory panel automatically
```

### Context Menu Missing Options

**Cause**: `OnCanRun` returns false or function not defined properly
**Fix**: Check item function definition and conditions

```lua
-- WRONG
ITEM.functions.use = function()
    -- Won't appear in menu!
end

-- CORRECT
ITEM.functions.use = {
    OnRun = function(item)
        -- Action code
    end,
    OnCanRun = function(item)
        return true  -- Always show
    end
}
```

### Items Overlapping

**Cause**: Grid collision detection bypassed or inventory state desync
**Fix**: Let framework handle positioning, don't manually set positions

```lua
-- WRONG
itemPanel:SetPos(100, 100)  -- Manual positioning breaks grid!

-- CORRECT
-- Let ixInventory handle positioning through SetInventory()
```

## See Also

- [Inventory System](../systems/inventory.md) - Backend inventory management
- [Items System](../systems/items.md) - Item definitions and functions
- [Derma Overview](derma-overview.md) - UI panel basics
- [Menus](menus.md) - Main menu integration
- [Storage Library](../libraries/storage.md) - Storage containers
- Source: `gamemode/core/derma/cl_inventory.lua`
