# Item System

The Helix item system provides a flexible, data-driven way to create and manage in-game items. Items can be anything from weapons and ammunition to consumables, clothing, and containers.

## Overview

### What is an Item?

An item in Helix is a table of data and functions that define:
- Display information (name, description, model, icon)
- Physical properties (width, height, weight)
- Behavior (what happens when used, equipped, dropped)
- Data storage (custom per-instance data)

### Item vs Entity

- **Item Definition**: Template registered with `ix.item.Register()` - defines what an item is
- **Item Instance**: A unique copy with its own ID and data - a specific item in the world
- **Item Entity**: Physical representation (`ix_item` entity) when dropped in world

```
Item Definition (Registered Template)
       ↓
Item Instance (In Inventory)
       ↓
Item Entity (Dropped in World)
```

## Item Structure

### Basic Item

**Minimum required structure:**

```lua
ITEM.name = "Example Item"
ITEM.description = "An example item"
ITEM.model = "models/props_junk/cardboard_box001a.mdl"
```

### Complete Item Structure

```lua
ITEM.name = "Complete Item"                    -- Display name
ITEM.description = "Full example"              -- Description shown in tooltip
ITEM.model = "models/props/model.mdl"          -- World model
ITEM.width = 1                                 -- Inventory width (grid cells)
ITEM.height = 1                                -- Inventory height (grid cells)
ITEM.category = "Miscellaneous"                -- Category for organization
ITEM.flag = "1"                                -- Permission flag required
ITEM.price = 50                                -- Default price for vendors
ITEM.rarity = "common"                         -- Rarity tier
ITEM.iconCam = {                               -- Custom icon camera position
    pos = Vector(0, 0, 200),
    ang = Angle(90, 0, 0),
    fov = 45
}

-- Functions
function ITEM:GetName()                        -- Dynamic name
    return self.name .. " (" .. (self:GetData("uses", 3)) .. " uses)"
end

function ITEM:GetDescription()                 -- Dynamic description
    return self.description .. "\nUses: " .. self:GetData("uses", 3)
end

function ITEM:OnInstanced(invID, x, y, item)   -- Called when item is created
    item:SetData("uses", 3)
    item:SetData("timestamp", os.time())
end

function ITEM:OnRemoved()                      -- Called when item is deleted
    print("Item removed:", self:GetID())
end

function ITEM:CanTransfer(oldInv, newInv)      -- Check if can move to new inventory
    return true
end

function ITEM:OnTransferred(oldInv, newInv)    -- Called when moved
    print("Item moved from", oldInv:GetID(), "to", newInv:GetID())
end
```

## Creating Items

### Item Registration

**File**: `schema/items/sh_itemname.lua` or `plugins/myplugin/items/sh_itemname.lua`

```lua
ITEM.name = "Health Kit"
ITEM.description = "Restores 50 health"
ITEM.model = "models/items/healthkit.mdl"
ITEM.width = 2
ITEM.height = 1
ITEM.category = "Medical"
ITEM.price = 100
```

Files in `items/` directories are automatically loaded and registered.

### Manual Registration

```lua
ix.item.Register("uniqueid", ITEM, false, nil, uniqueID, true)

-- Or using helper:
local ITEM = ix.item.New("uniqueid")
ITEM.name = "My Item"
-- ... set properties
ix.item.Register("uniqueid", ITEM)
```

### Base Items

Use base items to inherit functionality:

```lua
ITEM.name = "9mm Pistol"
ITEM.base = "base_weapons"  -- Inherit from weapon base
ITEM.weapon = "weapon_pistol"
ITEM.ammo = "pistol"
ITEM.width = 2
ITEM.height = 1
```

## Item Properties

### Display Properties

```lua
ITEM.name = "Display Name"           -- Shown in inventory
ITEM.description = "Text description"  -- Shown in tooltip
ITEM.model = "models/path/to/model.mdl"  -- 3D model
ITEM.skin = 0                        -- Model skin
ITEM.material = "materials/path.vmt"  -- Override material
```

### Physical Properties

```lua
ITEM.width = 1                       -- Inventory width (1-n grid cells)
ITEM.height = 1                      -- Inventory height (1-n grid cells)
ITEM.weight = 1                      -- Weight in kg (future use)
```

### Gameplay Properties

```lua
ITEM.category = "Weapons"            -- Organization category
ITEM.flag = "v"                      -- Required permission flag (single char)
ITEM.price = 100                     -- Default buy price
ITEM.rarity = "legendary"            -- Rarity (affects color, sorting)
ITEM.maxStack = 10                   -- Maximum stack size (not implemented)
```

### Icon Customization

```lua
-- Custom camera position for icon
ITEM.iconCam = {
    pos = Vector(0, 0, 200),         -- Camera position
    ang = Angle(90, 0, 0),           -- Camera angle
    fov = 45                         -- Field of view
}

-- Or override icon completely
ITEM.icon = "vgui/inventory/icon.png"
```

## Item Functions

### Lifecycle Hooks

```lua
function ITEM:OnInstanced(invID, x, y, item)
    -- Called when item instance is created
    -- Initialize custom data here
    item:SetData("condition", 100)
    item:SetData("owner", "")
end

function ITEM:OnRemoved()
    -- Called before item is deleted
    -- Clean up external references
end

function ITEM:OnRestored(item)
    -- Called when item is loaded from database
    -- Validate and fix data if needed
    if not item:GetData("condition") then
        item:SetData("condition", 100)
    end
end
```

### Inventory Hooks

```lua
function ITEM:CanTransfer(oldInventory, newInventory)
    -- Return false to prevent transfer
    -- Example: Prevent transfer if item is locked
    if self:GetData("locked") then
        return false
    end
    return true
end

function ITEM:OnTransferred(oldInventory, newInventory)
    -- Called after successful transfer
    local owner = self:GetOwner()
    if IsValid(owner) then
        owner:ChatPrint("Item moved!")
    end
end
```

### Display Hooks

```lua
function ITEM:GetName()
    -- Return dynamic name
    local condition = self:GetData("condition", 100)
    return self.name .. " (" .. condition .. "%)"
end

function ITEM:GetDescription()
    -- Return dynamic description
    local desc = self.description
    desc = desc .. "\nCondition: " .. self:GetData("condition", 100) .. "%"
    desc = desc .. "\nWeight: " .. (self.weight or 1) .. " kg"
    return desc
end

function ITEM:PopulateTooltip(tooltip)
    -- Customize tooltip panel
    local condition = self:GetData("condition", 100)

    local row = tooltip:AddRowAfter("name", "condition")
    row:SetText("Condition")
    row:SetBackgroundColor(Color(50, 50, 50))
    row:SizeToContents()
end

function ITEM:PaintOver(item, w, h)
    -- Draw on top of item icon
    -- Example: Draw durability bar
    local condition = item:GetData("condition", 100) / 100
    draw.RoundedBox(0, 4, h - 8, w - 8, 4, Color(0, 0, 0, 200))
    draw.RoundedBox(0, 5, h - 7, (w - 10) * condition, 2, Color(0, 255, 0))
end
```

## Item Data

### Setting Data

```lua
-- Set custom data on item instance
item:SetData("uses", 5)
item:SetData("owner", "STEAM_0:1:12345")
item:SetData("enchantment", "fire")
item:SetData("stats", {
    damage = 10,
    accuracy = 0.8
})
```

### Getting Data

```lua
-- Get data with default value
local uses = item:GetData("uses", 3)
local owner = item:GetData("owner", "")
local stats = item:GetData("stats", {})

-- Check if data exists
if item:GetData("enchantment") then
    -- Item has enchantment
end
```

### Synced Data

All item data is automatically synchronized between client and server.

### Data Persistence

Item data is automatically saved to the database when:
- Item is transferred
- Character disconnects
- Server shuts down
- Manual save is triggered

## Item Actions

### Defining Actions

```lua
-- Click action (primary button)
ITEM.functions.Use = {
    name = "Use",                    -- Button text
    icon = "icon16/tick.png",        -- Button icon
    OnRun = function(item)
        local client = item.player   -- Player who clicked

        if SERVER then
            client:SetHealth(math.min(client:Health() + 50, 100))
            client:EmitSound("items/medshot4.wav")
        end

        return true  -- Remove item after use (return false to keep)
    end,
    OnCanRun = function(item)
        -- Return false to disable button
        local client = item.player
        return client:Health() < 100
    end
}

-- Secondary action
ITEM.functions.Inspect = {
    name = "Inspect",
    icon = "icon16/magnifier.png",
    OnRun = function(item)
        local client = item.player
        client:ChatPrint("This item looks interesting...")
        return false  -- Don't remove item
    end,
    OnCanRun = function(item)
        return true  -- Always available
    end
}

-- Custom action with confirmation
ITEM.functions.Destroy = {
    name = "Destroy",
    icon = "icon16/bin.png",
    OnRun = function(item)
        return true  -- Remove item
    end,
    OnCanRun = function(item)
        return true
    end,
    bShowPrompt = true  -- Show confirmation prompt
}
```

### Multiple Actions

```lua
ITEM.functions.Use = { ... }
ITEM.functions.Equip = { ... }
ITEM.functions.Unequip = { ... }
ITEM.functions.Inspect = { ... }
ITEM.functions.Upgrade = { ... }
```

## Base Item Types

### base_weapons

Equippable weapons with ammunition.

```lua
ITEM.name = "Pistol"
ITEM.base = "base_weapons"
ITEM.weapon = "weapon_pistol"        -- Weapon class
ITEM.ammo = "pistol"                 -- Ammo type
ITEM.width = 2
ITEM.height = 1
```

**Features:**
- Equip/Unequip actions
- Ammunition tracking
- Weapon switching

### base_ammo

Ammunition for weapons.

```lua
ITEM.name = "Pistol Ammo"
ITEM.base = "base_ammo"
ITEM.ammo = "pistol"                 -- Ammo type
ITEM.ammoAmount = 30                 -- Rounds per box
ITEM.width = 1
ITEM.height = 1
```

**Features:**
- Load action (adds to ammo pool)
- Automatic weapon compatibility

### base_bags

Containers with internal inventory.

```lua
ITEM.name = "Backpack"
ITEM.base = "base_bags"
ITEM.width = 2
ITEM.height = 2
ITEM.invWidth = 5                    -- Internal inventory width
ITEM.invHeight = 5                   -- Internal inventory height
```

**Features:**
- Open action (shows internal inventory)
- Nested inventory system
- Cannot contain other bags by default

### base_outfit

Wearable clothing that changes playermodel.

```lua
ITEM.name = "Citizen Clothes"
ITEM.base = "base_outfit"
ITEM.outfitCategory = "torso"        -- Body part
ITEM.bodyGroups = {                  -- Bodygroup changes
    ["torso"] = 1
}
ITEM.replacements = {                -- Model replacements
    ["models/player/group01/male_01.mdl"] = "models/player/group01/male_02.mdl"
}
```

**Features:**
- Equip action (changes appearance)
- Bodygroup modifications
- Model replacements

### base_pacoutfit

PAC3 customization outfits.

```lua
ITEM.name = "Armor Outfit"
ITEM.base = "base_pacoutfit"
ITEM.pacData = "..."                 -- Encoded PAC data
```

**Features:**
- PAC3 integration
- Visual customization
- Saves PAC outfit data

## Item Methods

### Getting Item Information

```lua
-- Get item properties
local name = item:GetName()
local desc = item:GetDescription()
local model = item:GetModel()
local id = item:GetID()              -- Unique instance ID
local uniqueID = item.uniqueID       -- Item type ID

-- Get item owner
local client = item:GetOwner()       -- Returns player entity
local char = item:GetCharacter()     -- Returns character object
```

### Item Data Methods

```lua
-- Data methods
item:SetData(key, value)             -- Set custom data
local value = item:GetData(key, default)  -- Get custom data

-- Network data (visible to client)
item:SetNetVar(key, value)           -- Set networked variable
local value = item:GetNetVar(key, default)  -- Get networked variable
```

### Inventory Methods

```lua
-- Get item's inventory
local inv = item:GetInventory()

-- Get item position in inventory
local x, y = item:GetPosition()

-- Transfer to different inventory
item:Transfer(newInventoryID, x, y)  -- Returns true on success

-- Remove item
item:Remove()                        -- Deletes item permanently
```

### Spawning Item Entities

```lua
-- SERVER only
local entity = item:Spawn(position, angles)

-- Or manually:
local entity = ents.Create("ix_item")
entity:SetPos(position)
entity:SetAngles(angles)
entity:SetItem(item:GetID())
entity:Spawn()
```

## Creating Custom Item Types

### Simple Consumable

```lua
-- items/sh_healthkit.lua
ITEM.name = "Health Kit"
ITEM.description = "Restores 50 health"
ITEM.model = "models/items/healthkit.mdl"
ITEM.category = "Medical"
ITEM.price = 100

ITEM.functions.Use = {
    name = "Use",
    OnRun = function(item)
        local client = item.player

        if SERVER then
            local health = client:Health()
            local maxHealth = client:GetMaxHealth()

            if health >= maxHealth then
                client:NotifyLocalized("healthFull")
                return false  -- Don't remove item
            end

            client:SetHealth(math.min(health + 50, maxHealth))
            client:EmitSound("items/medshot4.wav")
        end

        return true  -- Remove item
    end,
    OnCanRun = function(item)
        return item.player:Health() < item.player:GetMaxHealth()
    end
}
```

### Durability Item

```lua
-- items/base_durable.lua (base type)
ITEM.name = "Durable Item"
ITEM.description = "A base for items with durability"
ITEM.maxDurability = 100

function ITEM:OnInstanced(invID, x, y, item)
    item:SetData("durability", self.maxDurability)
end

function ITEM:GetDurability()
    return self:GetData("durability", self.maxDurability)
end

function ITEM:SetDurability(amount)
    amount = math.Clamp(amount, 0, self.maxDurability)
    self:SetData("durability", amount)

    if amount <= 0 then
        self:Remove()  -- Destroy when broken
    end
end

function ITEM:DamageDurability(amount)
    self:SetDurability(self:GetDurability() - amount)
end

function ITEM:GetDescription()
    local desc = self.description
    desc = desc .. "\nDurability: " .. self:GetDurability() .. "/" .. self.maxDurability
    return desc
end

function ITEM:PaintOver(item, w, h)
    local percent = item:GetDurability() / self.maxDurability
    local barColor = Color(255 * (1 - percent), 255 * percent, 0)

    draw.RoundedBox(0, 4, h - 8, w - 8, 4, Color(0, 0, 0, 200))
    draw.RoundedBox(0, 5, h - 7, (w - 10) * percent, 2, barColor)
end

-- items/sh_pickaxe.lua (uses base)
ITEM.name = "Pickaxe"
ITEM.base = "base_durable"
ITEM.description = "A mining pickaxe"
ITEM.model = "models/props_c17/tools_wrench01a.mdl"
ITEM.maxDurability = 200

ITEM.functions.Mine = {
    name = "Mine",
    OnRun = function(item)
        local client = item.player

        if SERVER then
            client:ChatPrint("You mined some ore!")
            item:DamageDurability(10)
        end

        return false  -- Don't remove, durability handles destruction
    end
}
```

### Stackable Item

```lua
ITEM.name = "Gold Coin"
ITEM.description = "A valuable gold coin"
ITEM.model = "models/props/coin.mdl"
ITEM.maxStack = 99

function ITEM:OnInstanced(invID, x, y, item)
    item:SetData("stack", 1)
end

function ITEM:GetStack()
    return self:GetData("stack", 1)
end

function ITEM:SetStack(amount)
    self:SetData("stack", math.Clamp(amount, 1, self.maxStack))
end

function ITEM:GetName()
    local stack = self:GetStack()
    if stack > 1 then
        return self.name .. " (x" .. stack .. ")"
    end
    return self.name
end

function ITEM:CanTransfer(oldInv, newInv)
    -- Try to stack with existing items
    for _, item in pairs(newInv:GetItems()) do
        if item.uniqueID == self.uniqueID and item:GetID() != self:GetID() then
            local theirStack = item:GetStack()
            local myStack = self:GetStack()

            if theirStack + myStack <= self.maxStack then
                -- Can fully merge
                item:SetStack(theirStack + myStack)
                self:Remove()
                return false  -- Prevent transfer, we merged instead
            end
        end
    end

    return true  -- Allow transfer
end
```

## Item Spawning

### Giving Items to Players

```lua
-- SERVER only
local client = player.GetByID(1)
local character = client:GetCharacter()
local inventory = character:GetInventory()

-- Add item by unique ID
inventory:Add("example_item")

-- Add with callback
inventory:Add("example_item", 1, nil, nil, function(item)
    -- Item was added successfully
    item:SetData("custom", "value")
end)

-- Add to specific position
inventory:Add("example_item", 1, x, y)
```

### Spawning in World

```lua
-- SERVER only
ix.item.Spawn("example_item", Vector(0, 0, 0), function(item, entity)
    -- Item entity spawned
    entity:SetAngles(Angle(0, 90, 0))
end)

-- Or with instance
local item = ix.item.Instance(0, "example_item")
item:Spawn(Vector(0, 0, 0))
```

## Item Inventory Management

### Moving Items

```lua
-- Transfer between inventories
item:Transfer(newInventoryID, x, y, client)

-- Check if item can fit
if inventory:CanItemFit(item) then
    -- Item can be added
end

-- Find free position
local x, y = inventory:FindEmptySlot(item.width, item.height)
if x and y then
    -- Found position
end
```

### Removing Items

```lua
-- Remove item instance
item:Remove()

-- Remove from inventory (also deletes)
inventory:Remove(item:GetID())

-- Remove by unique ID (removes first match)
for _, item in pairs(inventory:GetItems()) do
    if item.uniqueID == "example_item" then
        item:Remove()
        break
    end
end
```

## Best Practices

### Do's

- ✅ Use item:SetData() for instance-specific data
- ✅ Check IsValid() on entities
- ✅ Validate player input on server
- ✅ Use appropriate base items
- ✅ Provide meaningful descriptions
- ✅ Test with multiple players
- ✅ Handle edge cases (nil values, invalid data)
- ✅ Use OnCanRun to disable invalid actions

### Don'ts

- ❌ Don't modify ITEM table at runtime
- ❌ Don't trust client-provided item data
- ❌ Don't create items without checking inventory space
- ❌ Don't forget to check realm (SERVER/CLIENT)
- ❌ Don't use item properties for instance data
- ❌ Don't create circular bag references
- ❌ Don't forget to return true/false in OnRun

## See Also

- [Inventory System](inventory.md)
- [Character System](character.md)
- [Item API Reference](../api/item.md)
- [Creating Items Guide](../guides/custom-items.md)
