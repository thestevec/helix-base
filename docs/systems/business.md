# Business & Shipment System

> **Reference**: `gamemode/core/libs/sh_business.lua`, `gamemode/core/derma/cl_business.lua`

The business system provides a bulk item purchasing interface where players can buy multiple items at once and receive them in a physical shipment crate. This system is ideal for roleplay servers where players need to order supplies, equipment, or goods.

## ⚠️ Important: Use Built-in Helix Business System

**Always use Helix's business/shipment system** rather than creating custom purchasing menus. The framework provides:
- Built-in UI for browsing and purchasing items
- Automatic inventory categorization and search
- Physical shipment crate spawning with timer
- Money transaction validation and logging
- Rate limiting to prevent spam
- Integration with character ownership
- Hook system for custom restrictions

## Core Concepts

### What is the Business System?

The business system consists of two main components:

1. **Business Menu**: A UI accessible from the F1 menu where players browse items by category, add them to a cart, and purchase in bulk
2. **Shipment Crate**: A physical entity (`ix_shipment`) that spawns when a purchase is made, containing the ordered items

### How It Works

1. Player opens business menu (F1 → Business tab)
2. Browses items by category or searches by name
3. Clicks items to add to cart (up to 10 of each item)
4. Clicks checkout to see total cost
5. Confirms purchase if they have enough money
6. Shipment crate spawns at player's feet
7. Player opens crate and extracts items to inventory or drops them
8. Crate auto-removes after 120 seconds or when empty

### Item Eligibility

Items appear in the business menu if:
- Item has a `price` property defined
- Item is not marked with `noBusiness = true`
- `ix.config.Get("allowBusiness")` is true (enabled by default)
- Custom `CanPlayerUseBusiness` hook doesn't block it

## Configuration

### Server Configuration

**Reference**: `gamemode/core/hooks/sh_hooks.lua:126`

Enable or disable the entire business system:

```lua
-- In schema/sh_config.lua or through admin panel
ix.config.Set("allowBusiness", true)  -- Enable (default)
ix.config.Set("allowBusiness", false) -- Disable completely
```

### Making Items Purchasable

**Set Item Price**:

```lua
-- items/weapons/sh_pistol.lua
ITEM.name = "Pistol"
ITEM.description = "A standard pistol"
ITEM.model = "models/weapons/w_pist_glock18.mdl"
ITEM.price = 500  -- Appears in business menu for $500
ITEM.category = "Weapons"
```

**Exclude Item from Business**:

```lua
-- items/special/sh_admin_item.lua
ITEM.name = "Admin Tool"
ITEM.price = 1000
ITEM.noBusiness = true  -- Won't appear in business menu
ITEM.category = "Special"
```

**No Price = Not Purchasable**:

```lua
-- items/quest/sh_quest_item.lua
ITEM.name = "Quest Item"
-- No price property = won't appear in business menu
```

## Hooks

### GM:CanPlayerUseBusiness

**Reference**: `gamemode/core/hooks/sh_hooks.lua:126`

Controls whether a player can purchase a specific item through the business menu.

```lua
-- Allow only certain factions to buy weapons
function SCHEMA:CanPlayerUseBusiness(client, uniqueID)
    local itemTable = ix.item.list[uniqueID]

    -- Block if player has no character
    if (!client:GetCharacter()) then
        return false
    end

    -- Restrict weapons to police faction
    if (itemTable.category == "Weapons") then
        local faction = client:GetCharacter():GetFaction()
        if (faction != FACTION_POLICE) then
            client:Notify("Only police can purchase weapons")
            return false
        end
    end

    -- Allow by default (or fall through to default checks)
end

-- Restrict by class
function PLUGIN:CanPlayerUseBusiness(client, uniqueID)
    local character = client:GetCharacter()
    local class = character:GetClass()

    -- Only merchants can use business menu
    if (class != CLASS_MERCHANT) then
        return false
    end
end

-- Restrict specific items
function PLUGIN:CanPlayerUseBusiness(client, uniqueID)
    -- Block expensive items for new players
    local itemTable = ix.item.list[uniqueID]
    local playtime = client:GetTimePlayed()

    if (itemTable.price and itemTable.price > 1000 and playtime < 3600) then
        client:Notify("You need 1 hour playtime to buy expensive items")
        return false
    end
end
```

**Parameters**:
- `client` (Player): The player attempting to purchase
- `uniqueID` (string): Unique ID of the item being checked

**Returns**:
- `false`: Block this item from appearing/being purchased
- `nil` or no return: Allow (runs default checks)

**Default Behavior**:
- Returns false if `ix.config.Get("allowBusiness")` is false
- Returns false if player has no character
- Returns false if item has `noBusiness = true`

### GM:CreateShipment

**Reference**: `gamemode/core/libs/sh_business.lua:71`

Called when a shipment crate is created.

```lua
function PLUGIN:CreateShipment(client, entity)
    -- Log shipment creation
    ix.log.Add(client, "shipmentCreated", entity:GetPos())

    -- Notify nearby players
    for _, v in ipairs(ents.FindInSphere(entity:GetPos(), 512)) do
        if (v:IsPlayer() and v != client) then
            v:Notify(client:Name() .. " received a shipment")
        end
    end

    -- Add custom effects
    local effectData = EffectData()
    effectData:SetOrigin(entity:GetPos())
    util.Effect("Explosion", effectData)
end
```

### GM:ShipmentItemTaken

**Reference**: `gamemode/core/libs/sh_business.lua:113`

Called when a player takes an item from a shipment.

```lua
function PLUGIN:ShipmentItemTaken(client, uniqueID, amount)
    local itemTable = ix.item.list[uniqueID]

    -- Log item extraction
    ix.log.Add(client, "shipmentTaken", itemTable:GetName(), amount)

    -- Custom notification
    client:Notify("Extracted: " .. itemTable:GetName())
end
```

### GM:CanPlayerOpenShipment

**Reference**: `entities/entities/ix_shipment.lua:42`

Controls whether a player can open a shipment crate.

```lua
function PLUGIN:CanPlayerOpenShipment(client, entity)
    local ownerID = entity:GetNetVar("owner", 0)
    local character = client:GetCharacter()

    -- Allow owner and faction members
    if (character:GetID() == ownerID) then
        return true  -- Owner can always open
    end

    -- Check if same faction
    local ownerChar = ix.char.loaded[ownerID]
    if (ownerChar and ownerChar:GetFaction() == character:GetFaction()) then
        return true  -- Faction members can open
    end

    client:Notify("This isn't your shipment!")
    return false
end
```

### GM:BuildBusinessMenu

**Reference**: `gamemode/core/derma/cl_business.lua:498`

Controls whether the business tab appears in the F1 menu.

```lua
-- Disable business menu entirely (client-side)
function PLUGIN:BuildBusinessMenu()
    return false  -- Don't show business tab
end

-- Show only for certain characters (client-side)
function PLUGIN:BuildBusinessMenu()
    local character = LocalPlayer():GetCharacter()
    if (!character) then
        return false
    end

    local class = character:GetClass()
    if (class != CLASS_MERCHANT) then
        return false  -- Hide menu for non-merchants
    end
end
```

## Complete Examples

### Example 1: Creating Purchasable Items

```lua
-- items/food/sh_apple.lua
ITEM.name = "Apple"
ITEM.description = "A fresh, red apple"
ITEM.model = "models/props/cs_italy/orange.mdl"
ITEM.price = 5  -- $5 dollars
ITEM.category = "Food"

function ITEM:OnUse()
    local client = self.player
    client:SetHealth(math.min(client:Health() + 10, client:GetMaxHealth()))
end

-- items/food/sh_water.lua
ITEM.name = "Water Bottle"
ITEM.description = "Bottled water"
ITEM.model = "models/props_junk/PopCan01a.mdl"
ITEM.price = 3
ITEM.category = "Food"

-- items/weapons/sh_pistol.lua
ITEM.name = "Pistol"
ITEM.model = "models/weapons/w_pist_glock18.mdl"
ITEM.price = 500
ITEM.category = "Weapons"
ITEM.class = "weapon_glock2"
```

### Example 2: Faction-Restricted Business

```lua
-- schema/sv_hooks.lua
function SCHEMA:CanPlayerUseBusiness(client, uniqueID)
    local character = client:GetCharacter()
    local itemTable = ix.item.list[uniqueID]

    -- Weapons only for law enforcement
    if (itemTable.category == "Weapons") then
        local faction = character:GetFaction()

        if (faction == FACTION_POLICE or faction == FACTION_SWAT) then
            return true
        else
            client:Notify("Only law enforcement can purchase weapons")
            return false
        end
    end

    -- Medical supplies only for medics
    if (itemTable.category == "Medical") then
        local class = character:GetClass()

        if (class != CLASS_MEDIC) then
            client:Notify("Only medics can purchase medical supplies")
            return false
        end
    end
end
```

### Example 3: Rank-Based Item Access

```lua
-- plugins/ranks/sh_hooks.lua
function PLUGIN:CanPlayerUseBusiness(client, uniqueID)
    local character = client:GetCharacter()
    local itemTable = ix.item.list[uniqueID]
    local rank = character:GetData("rank", 1)

    -- Check item rank requirement
    if (itemTable.requiredRank and rank < itemTable.requiredRank) then
        client:Notify("You need rank " .. itemTable.requiredRank .. " to purchase this")
        return false
    end
end

-- items/weapons/sh_assault_rifle.lua
ITEM.name = "Assault Rifle"
ITEM.price = 2000
ITEM.category = "Weapons"
ITEM.requiredRank = 3  -- Custom property
```

### Example 4: Limited Stock System

```lua
-- plugins/limitedstock/sv_plugin.lua
PLUGIN.stock = PLUGIN.stock or {}

function PLUGIN:CanPlayerUseBusiness(client, uniqueID)
    local stock = self.stock[uniqueID] or 0

    if (stock <= 0) then
        client:Notify("This item is out of stock!")
        return false
    end
end

function PLUGIN:CreateShipment(client, entity)
    -- Decrease stock for each item in shipment
    for uniqueID, amount in pairs(entity.items) do
        self.stock[uniqueID] = (self.stock[uniqueID] or 100) - amount
    end
end

-- Admin command to restock
ix.command.Add("RestockBusiness", {
    adminOnly = true,
    OnRun = function(self, client)
        PLUGIN.stock = {}
        client:Notify("Business restocked!")
    end
})
```

### Example 5: Custom Shipment Spawn Position

```lua
-- schema/sv_hooks.lua
function SCHEMA:CreateShipment(client, entity)
    -- Move shipment to designated delivery area
    local deliveryZone = ents.FindByName("delivery_zone")[1]

    if (IsValid(deliveryZone)) then
        entity:SetPos(deliveryZone:GetPos() + Vector(0, 0, 50))
        client:Notify("Your shipment is at the delivery zone!")
    end
end
```

### Example 6: Bulk Discount System

```lua
-- plugins/discounts/sh_hooks.lua
-- Modify business UI to calculate discounts (would need UI modification)

function PLUGIN:CreateShipment(client, entity)
    -- Count total items
    local totalItems = 0
    for _, amount in pairs(entity.items) do
        totalItems = totalItems + amount
    end

    -- Give partial refund for bulk orders
    if (totalItems >= 20) then
        local character = client:GetCharacter()
        local discount = math.floor(totalCost * 0.1)  -- 10% discount

        character:GiveMoney(discount)
        client:Notify("Bulk discount applied: " .. ix.currency.Get(discount))
    end
end
```

## Best Practices

### ✅ DO

- Set reasonable prices on items using the `price` property
- Use categories to organize items logically
- Use `CanPlayerUseBusiness` hook for faction/class restrictions
- Mark non-purchasable items with `noBusiness = true`
- Let shipments spawn naturally (don't prevent spawning)
- Allow the 120-second timer to work as designed
- Use hooks to log business transactions
- Test item prices for economic balance

### ❌ DON'T

- Don't create custom business menus - extend the existing one
- Don't forget to set `price` on items you want purchasable
- Don't make all items purchasable - some should be found/crafted
- Don't bypass money checks in `CanPlayerUseBusiness`
- Don't allow purchasing items that would break game balance
- Don't modify shipment contents after creation (anti-cheat risk)
- Don't set extremely high prices without testing economy
- Don't block rate limiting (prevents purchase spam)

**⚠️ Do NOT**:
```lua
-- WRONG: Creating custom business menu
function MyCustomBusinessMenu()
    -- Don't recreate the entire business system!
end

-- WRONG: Bypassing money checks
function PLUGIN:CanPlayerUseBusiness(client, uniqueID)
    return true  -- Always allow without checking anything!
end

-- WRONG: All items free
ITEM.price = 0  -- Makes items free (use nil instead to hide from menu)

-- CORRECT: Hide from menu if shouldn't be purchased
ITEM.price = 500  -- Purchasable for $500
-- Or
ITEM.noBusiness = true  -- Not purchasable, hide from menu
```

## Common Patterns

### Pattern 1: Category-Based Restrictions

```lua
function SCHEMA:CanPlayerUseBusiness(client, uniqueID)
    local itemTable = ix.item.list[uniqueID]
    local character = client:GetCharacter()
    local class = character:GetClass()

    -- Map categories to allowed classes
    local restrictions = {
        ["Weapons"] = {CLASS_POLICE, CLASS_SWAT},
        ["Medical"] = {CLASS_MEDIC, CLASS_DOCTOR},
        ["Food"] = {},  -- Everyone can buy
        ["Tools"] = {CLASS_ENGINEER, CLASS_MECHANIC}
    }

    local allowedClasses = restrictions[itemTable.category]

    if (allowedClasses and #allowedClasses > 0) then
        if (!table.HasValue(allowedClasses, class)) then
            return false
        end
    end
end
```

### Pattern 2: Opening Shipment UI

Players automatically open shipments by using them (E key). The framework handles this:

**Reference**: `entities/entities/ix_shipment.lua:39-54`

```lua
-- This is handled automatically by the framework
-- Override only for custom behavior
function ENTITY:Use(activator)
    -- Framework code shows UI after interaction time
    activator:PerformInteraction(ix.config.Get("itemPickupTime", 0.5), self, function(client)
        if (client:GetCharacter():GetID() == self:GetNetVar("owner", 0)) then
            -- Opens shipment UI
            client.ixShipment = self
            -- Sends net message to open UI...
        end
    end)
end
```

### Pattern 3: Taking Items from Shipment

```lua
-- Players can extract items two ways:
-- 1. Take to inventory (click item)
-- 2. Drop on ground (right-click item)

-- This is handled automatically via network messages
-- See: gamemode/core/libs/sh_business.lua:77-123
```

## Common Issues

### Business Menu Not Appearing

**Cause**: `allowBusiness` config disabled or `BuildBusinessMenu` hook blocking it
**Fix**: Enable business system and check hooks

```lua
-- Check config
print(ix.config.Get("allowBusiness"))  -- Should be true

-- Check for blocking hooks
hook.Add("BuildBusinessMenu", "DebugBusiness", function()
    print("BuildBusinessMenu called")
    -- Don't return false here!
end)
```

### Items Not Showing in Menu

**Cause**: No price set, `noBusiness = true`, or hook blocking
**Fix**: Set price and check item properties

```lua
-- ❌ WRONG: Item won't appear
ITEM.name = "Hidden Item"
-- No price property

-- ✅ CORRECT: Item appears
ITEM.name = "Visible Item"
ITEM.price = 100

-- Check if item should appear
local item = ix.item.list["my_item"]
print("Has price:", item.price)  -- Should have value
print("No business:", item.noBusiness)  -- Should be nil/false
```

### Can't Open Shipment

**Cause**: Wrong character trying to open, or hook blocking
**Fix**: Ensure owner or allowed faction member is opening

```lua
-- Debug shipment ownership
function PLUGIN:CanPlayerOpenShipment(client, entity)
    local ownerID = entity:GetNetVar("owner", 0)
    local clientID = client:GetCharacter():GetID()

    print("Owner:", ownerID, "Client:", clientID)

    if (ownerID != clientID) then
        client:Notify("Not your shipment!")
        return false
    end
end
```

### Purchase Failing Silently

**Cause**: Not enough money, rate limit, or invalid items
**Fix**: Check money and timing

```lua
-- Debug purchase issues
function PLUGIN:CanPlayerUseBusiness(client, uniqueID)
    local character = client:GetCharacter()
    local money = character:GetMoney()
    local itemTable = ix.item.list[uniqueID]

    print(string.format("Player: %s, Money: %d, Item: %s, Price: %d",
        client:Name(), money, uniqueID, itemTable.price or 0))

    -- Check if can afford
    if (itemTable.price and money < itemTable.price) then
        client:Notify("Not enough money!")
    end
end
```

### Shipment Disappearing Too Fast

**Cause**: 120-second auto-removal timer
**Fix**: Items must be extracted within 2 minutes

**Reference**: `entities/entities/ix_shipment.lua:30-36`

```lua
-- Extend shipment lifetime (in entity code)
function ENT:Initialize()
    -- ... existing code ...

    -- Change from 120 to 300 seconds (5 minutes)
    self:SetDeliveryTime(CurTime() + 300)

    timer.Simple(300, function()
        if (IsValid(self)) then
            self:Remove()
        end
    end)
end
```

## Technical Details

### Rate Limiting

**Reference**: `gamemode/core/libs/sh_business.lua:10-13`

Purchases are rate-limited to prevent spam:
- 0.5 second cooldown between purchases
- Stored in `client.ixNextBusiness`

### Purchase Limits

- Maximum 10 of each item per purchase
- Validated on both client and server
- Items clamped to 0-10 range

### Shipment Ownership

- Shipment stores owner's character ID in `owner` netvar
- Only owner can open by default
- Tracked in character's `charEnts` variable

### Item Extraction

Items can be extracted in two ways:
1. **To Inventory**: Adds item directly to character inventory
2. **Drop**: Spawns item entity on ground near crate

## See Also

- [Currency System](currency.md) - Money formatting and transactions
- [Inventory System](inventory.md) - Item storage and management
- [Items](items.md) - Creating and configuring items
- Source: `gamemode/core/libs/sh_business.lua`
- Entity: `entities/entities/ix_shipment.lua`
- UI: `gamemode/core/derma/cl_business.lua`
- Shipment UI: `gamemode/core/derma/cl_shipment.lua`
