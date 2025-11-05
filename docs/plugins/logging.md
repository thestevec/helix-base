# Logging Plugin

> **Reference**: `plugins/logging.lua`

The Logging plugin registers common log types for tracking player actions, character events, item interactions, and administrative actions. Provides comprehensive server logging for accountability and debugging.

## ⚠️ Important: Use Built-in Log Types

**Always use `ix.log.Add()`** with registered log types. The framework provides:
- Automatic logging of common events
- Formatted log messages
- Flag-based categorization (DANGER, WARNING, SUCCESS, etc.)
- Integration with admin viewing tools

## Registered Log Types

### Player Events
- `connect` - Player joins server
- `disconnect` - Player leaves server (includes timeout detection)
- `playerHurt` - Player takes damage
- `playerDeath` - Player dies

### Character Events
- `charCreate` - Character created
- `charLoad` - Character loaded
- `charDelete` - Character deleted

### Item Events
- `itemAction` - Item interaction (use, drop, etc.)
- `itemDestroy` - Item destroyed
- `inventoryAdd` - Item added to inventory
- `inventoryRemove` - Item removed from inventory

### Economic Events
- `money` - Money gained/lost
- `buy` - Vendor purchase
- `buydoor` - Door purchased
- `selldoor` - Door sold
- `shipmentOrder` - Shipment ordered
- `shipmentTake` - Item taken from shipment

### Administrative Events
- `command` - Command used
- `cfgSet` - Config changed (FLAG_DANGER)
- `pluginLoaded` - Plugin enabled
- `pluginUnloaded` - Plugin disabled

## See Also

- [Admin System](../systems/admin.md) - Viewing logs
- Source: `plugins/logging.lua`
