# Vendor Entity (ix_vendor)

> **Reference**: `plugins/vendor/entities/entities/ix_vendor.lua`

The `ix_vendor` entity represents an NPC vendor that can buy and sell items to players. It's part of the vendor plugin and provides a complete trading system.

## ⚠️ Important: Use Vendor Plugin Features

**Always use the vendor plugin's built-in features** rather than creating custom vendor systems. The framework provides:
- Admin-spawnable vendor NPCs
- Configurable buy/sell modes
- Stock management system
- Faction/class restrictions
- Money system for vendors
- Persistent vendor data
- Complete trading UI

## Core Concepts

### What is a Vendor Entity?

A vendor entity (`ix_vendor`) is an interactive NPC that trades items with players. Vendors can:
- Sell items to players (vendor has items, player buys)
- Buy items from players (player has items, vendor buys)
- Do both buying and selling
- Have limited stock that replenishes
- Restrict trading to specific factions/classes
- Display custom welcome/rejection messages

Key features:
- Spawnable by admins only
- Persistent (saves to plugin data)
- Customizable appearance (model, skin, bodygroups)
- Speech bubble indicator (optional)
- Complete trading interface
- Access control system

### Entity Properties

**Network Variables** (lines 10-14):
- `NoBubble` (Bool): Hide the speech bubble indicator
- `DisplayName` (String): Vendor's display name
- `Description` (String): Vendor description

**Internal Properties** (lines 30-33):
- `items` (Table): Item trading data (price, stock, mode)
- `messages` (Table): Custom messages (welcome, leave, no trade)
- `factions` (Table): Allowed factions
- `classes` (Table): Allowed classes
- `money` (Number): Vendor's current money (nil = infinite)
- `scale` (Number): Sell-back price multiplier (default 0.5 = 50%)
- `receivers` (Table): Players currently viewing vendor

## Vendor Constants

**Reference**: `plugins/vendor/sh_plugin.lua:16-36`

### Trade Modes

```lua
-- Item data indices
VENDOR_PRICE = 1      -- Item's price
VENDOR_STOCK = 2      -- Current stock count
VENDOR_MODE = 3       -- Buy/sell mode
VENDOR_MAXSTOCK = 4   -- Maximum stock

-- Trade mode values
VENDOR_SELLANDBUY = 1  -- Vendor both buys and sells this item
VENDOR_SELLONLY = 2    -- Vendor only sells to player
VENDOR_BUYONLY = 3     -- Vendor only buys from player

-- Message types
VENDOR_WELCOME = 1     -- Greeting when opening vendor
VENDOR_LEAVE = 2       -- Message when closing (unused)
VENDOR_NOTRADE = 3     -- Message when access denied
```

## Spawning Vendors

### Admin Spawning

Vendors are spawned using the entity tool with admin access:

```lua
-- Spawn via entity tool
-- 1. Open spawn menu
-- 2. Navigate to Entities > Helix > Vendor
-- 3. Click to place vendor
-- 4. Faces toward player automatically

-- SpawnFunction is called automatically
function ENT:SpawnFunction(client, trace)
    -- Creates vendor facing player
    -- Saves vendor data
end
```

### Programmatic Spawning

**Example**:

```lua
-- Create a vendor in code
local vendor = ents.Create("ix_vendor")
vendor:SetPos(Vector(100, 200, 50))
vendor:SetAngles(Angle(0, 90, 0))
vendor:Spawn()

-- Configure vendor
vendor:SetDisplayName("Merchant Bob")
vendor:SetDescription("General goods trader")
vendor:SetModel("models/humans/group01/male_01.mdl")

-- Set up items for sale
vendor.items = {
    ["item_medkit"] = {
        [VENDOR_PRICE] = 500,
        [VENDOR_STOCK] = 10,
        [VENDOR_MODE] = VENDOR_SELLONLY,
        [VENDOR_MAXSTOCK] = 10
    },
    ["ammo_pistol"] = {
        [VENDOR_PRICE] = 100,
        [VENDOR_STOCK] = 50,
        [VENDOR_MODE] = VENDOR_SELLANDBUY,
        [VENDOR_MAXSTOCK] = 50
    }
}

-- Give vendor money (or leave nil for infinite)
vendor.money = 10000

-- Set sell-back scale (50% of item price)
vendor.scale = 0.5

-- Save vendor data
PLUGIN:SaveData()
```

**⚠️ Do NOT**:
```lua
-- WRONG: Creating vendor without proper setup
local vendor = ents.Create("ix_vendor")
vendor:Spawn()  -- No items, no name, no configuration!

-- WRONG: Invalid item data structure
vendor.items = {
    ["item_medkit"] = 500  -- Should be table with VENDOR_* indices!
}
```

## Configuring Vendors

### Item Configuration

**Reference**: `plugins/vendor/entities/entities/ix_vendor.lua:91-146`

```lua
-- Configure item for vendor
vendor.items[uniqueID] = {
    [VENDOR_PRICE] = price,        -- Sale price
    [VENDOR_STOCK] = current,      -- Current stock (optional)
    [VENDOR_MODE] = mode,          -- VENDOR_SELLANDBUY, VENDOR_SELLONLY, or VENDOR_BUYONLY
    [VENDOR_MAXSTOCK] = max        -- Max stock (optional, enables stock system)
}
```

**Examples**:

```lua
-- Item vendor sells (infinite stock)
vendor.items["weapon_pistol"] = {
    [VENDOR_PRICE] = 1500,
    [VENDOR_MODE] = VENDOR_SELLONLY
}

-- Item vendor buys from players
vendor.items["scrap_metal"] = {
    [VENDOR_PRICE] = 25,
    [VENDOR_MODE] = VENDOR_BUYONLY
}

-- Item vendor both buys and sells (with stock)
vendor.items["food_bread"] = {
    [VENDOR_PRICE] = 50,
    [VENDOR_STOCK] = 20,
    [VENDOR_MODE] = VENDOR_SELLANDBUY,
    [VENDOR_MAXSTOCK] = 20
}
```

### Access Restrictions

**Reference**: `plugins/vendor/entities/entities/ix_vendor.lua:67-89`

```lua
-- Restrict to specific factions
vendor.factions = {
    ["citizen"] = true,
    ["police"] = true
}

-- Further restrict to specific classes within allowed factions
vendor.classes = {
    ["medic"] = true,
    ["engineer"] = true
}

-- Check access
function ENT:CanAccess(client)
    -- Returns true if player's faction and class match
end
```

**Examples**:

```lua
-- Police-only weapon vendor
local weaponVendor = ents.Create("ix_vendor")
weaponVendor:Spawn()
weaponVendor:SetDisplayName("Police Armory")
weaponVendor.factions = {
    ["police"] = true
}
weaponVendor.items = {
    ["weapon_pistol"] = {
        [VENDOR_PRICE] = 2000,
        [VENDOR_MODE] = VENDOR_SELLONLY
    }
}

-- Public vendor (no restrictions)
local generalVendor = ents.Create("ix_vendor")
generalVendor:Spawn()
generalVendor:SetDisplayName("General Store")
vendor.factions = {}  -- Empty = everyone can access
```

### Custom Messages

**Reference**: `plugins/vendor/entities/entities/ix_vendor.lua:192-204`

```lua
-- Set custom messages
vendor.messages = {
    [VENDOR_WELCOME] = "Welcome to my shop!",
    [VENDOR_NOTRADE] = "I don't trade with your kind."
}
```

## Trading System

### Selling to Players

**Reference**: `plugins/vendor/entities/entities/ix_vendor.lua:108-128`

When a player buys an item:

```lua
function ENT:CanSellToPlayer(client, uniqueID)
    -- Checks:
    -- 1. Item exists and has data
    -- 2. Mode is not VENDOR_BUYONLY
    -- 3. Player has enough money
    -- 4. Vendor has stock (if stock enabled)
end
```

**Purchase flow**:
1. Player clicks "Buy" on item
2. Framework checks `CanSellToPlayer()`
3. Takes money from player
4. Adds item to player's inventory
5. Gives money to vendor (if vendor uses money)
6. Decrements vendor stock (if stock enabled)
7. Logs transaction

### Buying from Players

**Reference**: `plugins/vendor/entities/entities/ix_vendor.lua:130-146`

When a player sells an item:

```lua
function ENT:CanBuyFromPlayer(client, uniqueID)
    -- Checks:
    -- 1. Item exists and has data
    -- 2. Mode is VENDOR_SELLONLY
    -- 3. Vendor has enough money
end
```

**Sell flow**:
1. Player clicks "Sell" on item
2. Framework checks `CanBuyFromPlayer()`
3. Removes item from player's inventory
4. Calculates sell price (item price * vendor.scale)
5. Gives money to player
6. Takes money from vendor (if vendor uses money)
7. Increments vendor stock (if stock enabled)
8. Logs transaction

### Money System

**Reference**: `plugins/vendor/entities/entities/ix_vendor.lua:235-283`

```lua
-- Set vendor money (finite money pool)
vendor:SetMoney(5000)

-- Give vendor infinite money
vendor.money = nil

-- Check if vendor can afford
vendor:HasMoney(amount)

-- Add/remove money
vendor:GiveMoney(amount)
vendor:TakeMoney(amount)

-- Get vendor's money
local money = vendor:GetMoney()
```

### Stock Management

**Reference**: `plugins/vendor/entities/entities/ix_vendor.lua:255-283`

```lua
-- Get stock info
local current, max = vendor:GetStock(uniqueID)

-- Set stock
vendor:SetStock(uniqueID, amount)  -- Clamped to maxstock

-- Add to stock
vendor:AddStock(uniqueID, 5)  -- Adds 5

-- Remove from stock
vendor:TakeStock(uniqueID, 3)  -- Removes 3
```

## Hooks and Events

### CanPlayerUseVendor

```lua
-- Control who can open vendors
hook.Add("CanPlayerUseVendor", "CustomVendorAccess", function(client, vendor)
    local character = client:GetCharacter()

    -- Example: Require license
    if not character:GetData("tradingLicense", false) then
        client:Notify("You need a trading license!")
        return false
    end

    -- Example: Check cooldown
    if character:GetData("vendorCooldown", 0) > CurTime() then
        client:Notify("You must wait before trading again!")
        return false
    end
end)
```

### Logging

**Reference**: `plugins/vendor/sh_plugin.lua:128-143`

```lua
-- Vendor usage is automatically logged:
ix.log.AddType("vendorUse")   -- When vendor is opened
ix.log.AddType("vendorBuy")   -- When item is purchased
ix.log.AddType("vendorSell")  -- When item is sold
```

## Client-Side Features

### Speech Bubble

**Reference**: `plugins/vendor/entities/entities/ix_vendor.lua:285-311`

```lua
-- Vendors show a floating speech bubble above them
-- Can be disabled with:
vendor:SetNoBubble(true)
```

### Tooltip

**Reference**: `plugins/vendor/entities/entities/ix_vendor.lua:331-344`

```lua
function ENT:OnPopulateEntityInfo(container)
    -- Shows vendor's display name
    -- Shows description (if set)
end
```

## Complete Examples

### Basic General Store

```lua
-- Create a general store vendor
local vendor = ents.Create("ix_vendor")
vendor:SetPos(Vector(500, 300, 64))
vendor:SetAngles(Angle(0, 180, 0))
vendor:Spawn()

-- Appearance
vendor:SetModel("models/humans/group01/male_07.mdl")
vendor:SetDisplayName("General Merchant")
vendor:SetDescription("Selling basic supplies")

-- Items for sale
vendor.items = {
    ["food_bread"] = {
        [VENDOR_PRICE] = 50,
        [VENDOR_MODE] = VENDOR_SELLONLY
    },
    ["food_water"] = {
        [VENDOR_PRICE] = 25,
        [VENDOR_MODE] = VENDOR_SELLONLY
    },
    ["item_medkit"] = {
        [VENDOR_PRICE] = 500,
        [VENDOR_STOCK] = 5,
        [VENDOR_MODE] = VENDOR_SELLONLY,
        [VENDOR_MAXSTOCK] = 5
    }
}

-- Infinite money
vendor.money = nil

-- Save
PLUGIN:SaveData()
```

### Scrap Dealer (Buying Items)

```lua
-- Vendor that only buys items from players
local dealer = ents.Create("ix_vendor")
dealer:SetPos(Vector(600, 400, 64))
dealer:Spawn()

dealer:SetModel("models/humans/group01/male_04.mdl")
dealer:SetDisplayName("Scrap Dealer")
dealer:SetDescription("I buy junk")

-- Items vendor will buy
dealer.items = {
    ["scrap_metal"] = {
        [VENDOR_PRICE] = 100,  -- Pays $100 for scrap
        [VENDOR_MODE] = VENDOR_BUYONLY
    },
    ["scrap_electronics"] = {
        [VENDOR_PRICE] = 150,
        [VENDOR_MODE] = VENDOR_BUYONLY
    }
}

-- Limited money
dealer.money = 5000

-- Custom messages
dealer.messages = {
    [VENDOR_WELCOME] = "Got any scrap for me?",
    [VENDOR_NOTRADE] = "I don't deal with you."
}

PLUGIN:SaveData()
```

### Faction-Restricted Vendor

```lua
-- Police armory vendor
local armory = ents.Create("ix_vendor")
armory:SetPos(Vector(700, 500, 64))
armory:Spawn()

armory:SetModel("models/humans/group01/male_02.mdl")
armory:SetDisplayName("Police Quartermaster")

-- Restrict to police faction
armory.factions = {
    ["police"] = true
}

-- Weapons for police
armory.items = {
    ["weapon_pistol"] = {
        [VENDOR_PRICE] = 1500,
        [VENDOR_MODE] = VENDOR_SELLONLY
    },
    ["weapon_rifle"] = {
        [VENDOR_PRICE] = 3000,
        [VENDOR_STOCK] = 2,
        [VENDOR_MODE] = VENDOR_SELLONLY,
        [VENDOR_MAXSTOCK] = 2
    },
    ["ammo_pistol"] = {
        [VENDOR_PRICE] = 100,
        [VENDOR_MODE] = VENDOR_SELLONLY
    }
}

armory.messages = {
    [VENDOR_NOTRADE] = "This armory is for police officers only."
}

PLUGIN:SaveData()
```

### Restocking Vendor

```lua
-- Vendor that restocks items over time
hook.Add("Think", "VendorRestock", function()
    for _, vendor in ipairs(ents.FindByClass("ix_vendor")) do
        if not vendor.nextRestock or vendor.nextRestock < CurTime() then
            -- Restock every 5 minutes
            vendor.nextRestock = CurTime() + 300

            for uniqueID, data in pairs(vendor.items) do
                if data[VENDOR_MAXSTOCK] then
                    -- Restock to max
                    vendor:SetStock(uniqueID, data[VENDOR_MAXSTOCK])
                end
            end
        end
    end
end)
```

### Dynamic Pricing

```lua
-- Adjust prices based on demand
hook.Add("VendorBuy", "DynamicPricing", function(client, vendor, itemID, price)
    local data = vendor.items[itemID]

    if data then
        -- Increase price by 10% when sold
        local newPrice = math.floor(data[VENDOR_PRICE] * 1.1)
        data[VENDOR_PRICE] = newPrice

        PLUGIN:SaveData()
    end
end)

hook.Add("VendorSell", "DynamicPricing", function(client, vendor, itemID, price)
    local data = vendor.items[itemID]

    if data then
        -- Decrease price by 5% when bought from player
        local newPrice = math.floor(data[VENDOR_PRICE] * 0.95)
        data[VENDOR_PRICE] = math.max(newPrice, 1)

        PLUGIN:SaveData()
    end
end)
```

## Best Practices

### ✅ DO

- Use the vendor editor menu (right-click vendor as admin)
- Save vendor data after programmatic changes
- Validate item existence before adding to vendor
- Use stock system for limited items
- Set appropriate sell-back scales (default 0.5 = 50%)
- Use faction/class restrictions appropriately
- Give vendors descriptive names
- Set custom welcome messages

### ❌ DON'T

- Don't forget to call `PLUGIN:SaveData()` after creating vendors
- Don't set negative prices
- Don't bypass `CanSellToPlayer`/`CanBuyFromPlayer` checks
- Don't create vendors without items
- Don't forget to set `VENDOR_MODE` for items
- Don't use extremely high stock values (performance)
- Don't modify vendor data on client side

## Common Patterns

### Vendor That Gives Quest Rewards

```lua
-- Vendor that only sells to players who completed quest
hook.Add("CanPlayerUseVendor", "QuestVendor", function(client, vendor)
    if vendor:GetDisplayName() == "Quest Master" then
        local character = client:GetCharacter()

        if not character:GetData("questCompleted", false) then
            client:Notify("Complete the quest first!")
            return false
        end
    end
end)
```

### Bartering Vendor (No Money)

```lua
-- Vendor that trades items for items (no money)
-- This requires custom implementation beyond basic vendor
-- You'd need to modify the vendor plugin or create a custom system
```

## Common Issues

### Vendor Doesn't Save

**Cause**: Plugin data not saved after vendor creation
**Fix**: Call `PLUGIN:SaveData()` after creating/modifying vendors

```lua
local vendor = ents.Create("ix_vendor")
vendor:Spawn()
-- Configure vendor...
PLUGIN:SaveData()  -- Important!
```

### Can't Access Vendor

**Cause**: Faction/class restrictions or hook blocking
**Fix**: Check `vendor.factions` and `vendor.classes`

```lua
-- Debug access
print("Factions:", vendor.factions)
print("Classes:", vendor.classes)
print("Can access:", vendor:CanAccess(client))
```

### Stock Doesn't Work

**Cause**: Missing `VENDOR_MAXSTOCK` in item data
**Fix**: Set both `VENDOR_STOCK` and `VENDOR_MAXSTOCK`

```lua
-- Correct stock setup
vendor.items[uniqueID] = {
    [VENDOR_PRICE] = 100,
    [VENDOR_STOCK] = 10,
    [VENDOR_MAXSTOCK] = 10  -- Required for stock system!
}
```

### Vendor Has Wrong Model After Restart

**Cause**: Model set after spawn but not saved
**Fix**: Set model before saving

```lua
vendor:SetModel("models/...")
PLUGIN:SaveData()  -- Saves model
```

## See Also

- [Item System](../systems/items.md) - Core item system
- [Faction System](../systems/factions.md) - Faction restrictions
- [Class System](../systems/classes.md) - Class restrictions
- [Currency Library](../libraries/currency.md) - Money system (if documented)
- Source: `plugins/vendor/entities/entities/ix_vendor.lua`
- Source: `plugins/vendor/sh_plugin.lua`
