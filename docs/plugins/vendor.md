# Vendor Plugin

> **Reference**: `plugins/vendor/`

The Vendor plugin allows creation of NPC vendors that can buy and sell items. Vendors have customizable stock, prices, faction restrictions, and can be placed anywhere on the map with full persistence.

## ⚠️ Important: Use Built-in Vendor System

**Always spawn vendors through the vendor entity**. The framework provides:
- Stock management with max limits
- Buy/sell/both modes per item
- Faction and class restrictions
- Persistent vendor placement and configuration
- Visual editor interface

## Creating Vendors

1. Spawn `ix_vendor` entity via admin tools
2. Use physgun right-click to edit vendor
3. Configure name, description, items, factions
4. Vendors save automatically

## Vendor Modes

**Per Item**:
- `VENDOR_SELLANDBUY` (1) - Vendor buys and sells this item
- `VENDOR_SELLONLY` (2) - Vendor only sells (won't buy back)
- `VENDOR_BUYONLY` (3) - Vendor only buys (won't sell)

## Item Configuration

Each vendor item has:
- **Price**: Cost in currency
- **Stock**: Current available quantity
- **Mode**: Buy/sell/both
- **Max Stock**: Maximum inventory

## Faction Restrictions

Vendors can be restricted to:
- Specific factions
- Specific classes within factions
- Everyone (if no restrictions set)

## Editor Features

The vendor editor allows configuring:
- Display name
- Description
- Model and appearance
- Speech bubble toggle
- Faction/class access
- Item list with prices and stock
- Vendor starting money

## Developer Functions

```lua
-- Check if player can access vendor
if vendor:CanAccess(client) then
    -- Open vendor menu
end
```

## Permissions

**Privilege**: `Helix - Manage Vendors`
Required to edit vendors.

## See Also

- [Inventory System](../systems/inventory.md) - Item storage
- [Items System](../systems/items.md) - Creating items
- [Economy System](../systems/economy.md) - Currency
- Source: `plugins/vendor/`
