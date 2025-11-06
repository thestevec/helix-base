# Doors Plugin

> **Reference**: `plugins/doors/`

The Doors plugin provides a comprehensive door ownership and access control system. Players can buy/sell doors, manage access levels (owner, tenant, guest), lock/unlock doors, and share access with other characters.

## ⚠️ Important: Use Built-in Door System

**Always use door commands** for managing doors. The framework provides:
- Purchase/sell door system
- Three-tier access control (owner/tenant/guest)
- Door locking with configurable time
- Child door support (linked doors)
- Persistent door ownership

## Access Levels

- `DOOR_OWNER` (3) - Full control (can sell, manage access)
- `DOOR_TENANT` (2) - Can lock/unlock
- `DOOR_GUEST` (1) - Can use when unlocked
- `DOOR_NONE` (0) - No access

## Commands

- `/DoorBuy` - Purchase a door
- `/DoorSell` - Sell your door
- `/DoorLock` - Lock/unlock door
- `/DoorSetOwner` - Transfer ownership
- `/DoorAddTenant` - Add tenant
- `/DoorAddGuest` - Add guest

## Configuration

- `doorCost` (default: 10) - Door purchase price
- `doorSellRatio` (default: 0.5) - Refund percentage when selling
- `doorLockTime` (default: 1) - Time to lock/unlock doors

## See Also

- [Factions System](../systems/factions.md) - Faction-owned doors
- Source: `plugins/doors/`
