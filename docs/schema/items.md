# Creating Schema Items

> **Reference**: `gamemode/core/libs/sh_item.lua`, `schema/items/`

Items are the objects players interact with in your schema. This guide shows how to create items specifically for your schema.

## ⚠️ Important: Use Helix Item System

**Always use Helix's built-in item registration** rather than creating custom item systems. The framework provides:
- Automatic registration from `schema/items/` folder
- Database persistence
- Network synchronization
- Inventory integration
- Item actions and interactions

## Core Concepts

### What are Schema Items?

Schema items are unique to your roleplay setting:
- **HL2RP**: Citizen ID cards, rations, Combine equipment
- **DarkRP**: Job-specific tools, drugs, money printers
- **Medieval RP**: Swords, armor, food, crafting materials

Each item has:
- Visual representation (model, icon)
- Inventory size (width, height)
- Functions (use, equip, drop)
- Custom data (durability, charges, owner)

## Creating Items

### File Location

**Place item files** in:
```
schema/items/sh_itemname.lua
```

**File naming convention**:
- `sh_` prefix (shared realm)
- Descriptive item name
- `.lua` extension

Examples:
- `sh_cid.lua` → Citizen ID card
- `sh_ration.lua` → Food ration
- `sh_medkit.lua` → Medical kit

### Basic Item Template

```lua
-- schema/items/sh_medkit.lua
ITEM.name = "Medical Kit"
ITEM.description = "A kit containing medical supplies"
ITEM.model = "models/items/healthkit.mdl"
ITEM.width = 2
ITEM.height = 1
ITEM.category = "Medical"
ITEM.price = 100

ITEM.functions.Use = {
    name = "Use",
    OnRun = function(item)
        local client = item.player

        if SERVER then
            client:SetHealth(math.min(client:Health() + 50, 100))
            client:EmitSound("items/medshot4.wav")
        end

        return true  -- Remove item after use
    end,
    OnCanRun = function(item)
        return item.player:Health() < 100
    end
}
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't create custom item tables
CUSTOM_ITEMS = {}
CUSTOM_ITEMS["medkit"] = {...}

-- WRONG: Don't bypass item system
function GiveCustomItem(client)
    client.customItems = client.customItems or {}
    table.insert(client.customItems, {...})
end
```

## Required Properties

### Minimal Requirements

```lua
ITEM.name = "Item Name"                        -- REQUIRED
ITEM.description = "What the item does"        -- REQUIRED
ITEM.model = "models/path/to/model.mdl"        -- REQUIRED
```

**⚠️ If missing**: Item may not display properly or register correctly.

## Common Item Examples

### Citizen ID Card (HL2RP)

```lua
-- schema/items/sh_cid.lua
ITEM.name = "Citizen ID"
ITEM.description = "An identification card issued by the Combine"
ITEM.model = "models/gibs/metal_gib4.mdl"
ITEM.width = 1
ITEM.height = 1
ITEM.category = "Documents"

function ITEM:GetName()
    return self.name .. " #" .. (self:GetData("cidNumber", "00000"))
end

function ITEM:GetDescription()
    local description = self.description

    local cidNumber = self:GetData("cidNumber", "00000")
    local owner = self:GetData("owner", "Unknown")

    description = description .. "\n\nCID: #" .. cidNumber
    description = description .. "\nOwner: " .. owner

    return description
end

function ITEM:OnInstanced(invID, x, y, item)
    -- Generate random CID number
    item:SetData("cidNumber", string.format("%05d", math.random(1, 99999)))

    -- Set owner
    local character = item:GetOwner():GetCharacter()
    if character then
        item:SetData("owner", character:GetName())
    end
end

ITEM.functions.Show = {
    name = "Show",
    OnRun = function(item)
        local client = item.player
        local trace = client:GetEyeTrace()
        local target = trace.Entity

        if IsValid(target) and target:IsPlayer() and target:GetPos():Distance(client:GetPos()) <= 96 then
            local owner = item:GetData("owner", "Unknown")
            local cidNumber = item:GetData("cidNumber", "00000")

            target:ChatPrint(client:Name() .. " shows their ID:")
            target:ChatPrint("CID #" .. cidNumber .. " - " .. owner)
        else
            client:Notify("No one nearby to show")
        end

        return false  -- Don't remove item
    end
}
```

### Food Ration

```lua
-- schema/items/sh_ration.lua
ITEM.name = "Food Ration"
ITEM.description = "A standard Combine food supplement"
ITEM.model = "models/props_junk/garbage_metalcan001a.mdl"
ITEM.width = 1
ITEM.height = 1
ITEM.category = "Consumables"
ITEM.price = 20

ITEM.functions.Consume = {
    name = "Consume",
    icon = "icon16/cup.png",
    OnRun = function(item)
        local client = item.player

        if SERVER then
            client:SetHealth(math.min(client:Health() + 25, 100))
            client:EmitSound("npc/barnacle/barnacle_gulp2.wav")
            client:ChatPrint("You consumed the ration")

            -- Restore stamina
            local character = client:GetCharacter()
            character:SetData("stamina", 100)
        end

        return true  -- Remove item
    end
}
```

### Weapon Item

```lua
-- schema/items/sh_pistol.lua
ITEM.name = "9mm Pistol"
ITEM.description = "A standard semi-automatic pistol"
ITEM.model = "models/weapons/w_pistol.mdl"
ITEM.width = 2
ITEM.height = 1
ITEM.category = "Weapons"
ITEM.price = 500
ITEM.base = "base_weapons"  -- Inherit from weapon base
ITEM.weapon = "weapon_pistol"
ITEM.ammo = "pistol"

function ITEM:OnInstanced(invID, x, y, item)
    -- Set random condition
    item:SetData("condition", math.random(75, 100))
end

function ITEM:GetDescription()
    local description = self.description
    local condition = self:GetData("condition", 100)

    description = description .. "\n\nCondition: " .. condition .. "%"

    return description
end

function ITEM:PaintOver(item, w, h)
    -- Draw condition bar
    local condition = item:GetData("condition", 100) / 100
    local barColor = Color(255 * (1 - condition), 255 * condition, 0)

    draw.RoundedBox(0, 4, h - 8, w - 8, 4, Color(0, 0, 0, 200))
    draw.RoundedBox(0, 5, h - 7, (w - 10) * condition, 2, barColor)
end
```

### Crafting Material

```lua
-- schema/items/sh_scrap_metal.lua
ITEM.name = "Scrap Metal"
ITEM.description = "Pieces of salvaged metal"
ITEM.model = "models/gibs/metal_gib4.mdl"
ITEM.width = 1
ITEM.height = 1
ITEM.category = "Materials"
ITEM.price = 10

function ITEM:OnInstanced(invID, x, y, item)
    item:SetData("amount", 1)
end

function ITEM:GetName()
    local amount = self:GetData("amount", 1)
    if amount > 1 then
        return self.name .. " (x" .. amount .. ")"
    end
    return self.name
end

-- Stack similar items
function ITEM:CanTransfer(oldInv, newInv)
    -- Try to merge with existing stacks
    for _, item in pairs(newInv:GetItems()) do
        if item.uniqueID == self.uniqueID and item:GetID() != self:GetID() then
            local theirAmount = item:GetData("amount", 1)
            local myAmount = self:GetData("amount", 1)
            local maxStack = 99

            if theirAmount + myAmount <= maxStack then
                item:SetData("amount", theirAmount + myAmount)
                self:Remove()
                return false
            end
        end
    end

    return true
end
```

### Container Item

```lua
-- schema/items/sh_backpack.lua
ITEM.name = "Backpack"
ITEM.description = "A sturdy backpack for carrying items"
ITEM.model = "models/props_c17/suitcase001a.mdl"
ITEM.width = 2
ITEM.height = 2
ITEM.category = "Containers"
ITEM.price = 200
ITEM.base = "base_bags"  -- Inherit from bag base
ITEM.invWidth = 5
ITEM.invHeight = 4

function ITEM:OnInstanced(invID, x, y, item)
    -- Create internal inventory
    local inventory = ix.inventory.Create(self.invWidth, self.invHeight, item:GetID())
    inventory.vars.isBag = item:GetID()
    item:SetData("invID", inventory:GetID())
end
```

## Item Properties

### Display Properties

```lua
ITEM.name = "Display Name"                     -- Shown in inventory
ITEM.description = "What it does"              -- Shown in tooltip
ITEM.model = "models/path/to/model.mdl"        -- 3D model
ITEM.skin = 0                                  -- Model skin
ITEM.category = "Category"                     -- Organization
```

### Physical Properties

```lua
ITEM.width = 1                                 -- Inventory width
ITEM.height = 1                                -- Inventory height
ITEM.weight = 1                                -- Weight (future use)
```

### Gameplay Properties

```lua
ITEM.price = 100                               -- Default buy price
ITEM.flag = "v"                                -- Required flag
ITEM.faction = FACTION_POLICE                  -- Faction restriction
ITEM.class = CLASS_OFFICER                     -- Class restriction
```

## Item Functions

### Use Function

Most common item function:

```lua
ITEM.functions.Use = {
    name = "Use",                              -- Button text
    icon = "icon16/tick.png",                  -- Button icon
    OnRun = function(item)
        local client = item.player

        if SERVER then
            -- Do something
            client:SetHealth(math.min(client:Health() + 25, 100))
        end

        return true  -- Remove item (false = keep)
    end,
    OnCanRun = function(item)
        -- Return false to disable button
        return item.player:Health() < 100
    end
}
```

### Multiple Functions

Items can have multiple actions:

```lua
ITEM.functions.Equip = {
    name = "Equip",
    OnRun = function(item)
        -- Equip item
        return false
    end
}

ITEM.functions.Drop = {
    name = "Drop",
    OnRun = function(item)
        -- Drop item
        return false
    end
}

ITEM.functions.Destroy = {
    name = "Destroy",
    icon = "icon16/bin.png",
    OnRun = function(item)
        return true  -- Remove item
    end,
    bShowPrompt = true  -- Show confirmation
}
```

## Item Hooks

### OnInstanced

Called when item is first created:

```lua
function ITEM:OnInstanced(invID, x, y, item)
    -- Initialize custom data
    item:SetData("condition", 100)
    item:SetData("uses", 3)
    item:SetData("timestamp", os.time())

    -- Set owner
    local character = item:GetOwner():GetCharacter()
    if character then
        item:SetData("owner", character:GetName())
    end
end
```

### OnRemoved

Called before item is deleted:

```lua
function ITEM:OnRemoved()
    -- Clean up references
    print("Item removed:", self:GetID())

    -- If item has internal inventory, it's auto-deleted
end
```

### Dynamic Name and Description

```lua
function ITEM:GetName()
    local condition = self:GetData("condition", 100)
    return self.name .. " (" .. condition .. "%)"
end

function ITEM:GetDescription()
    local desc = self.description
    local uses = self:GetData("uses", 3)

    desc = desc .. "\n\nUses remaining: " .. uses
    desc = desc .. "\nCondition: " .. self:GetData("condition", 100) .. "%"

    return desc
end
```

### Custom Tooltip

```lua
function ITEM:PopulateTooltip(tooltip)
    local owner = self:GetData("owner")

    if owner then
        local row = tooltip:AddRowAfter("description", "owner")
        row:SetText("Owner: " .. owner)
        row:SetBackgroundColor(Color(50, 50, 50))
        row:SizeToContents()
    end
end
```

### Custom Icon Overlay

```lua
function ITEM:PaintOver(item, w, h)
    local uses = item:GetData("uses", 3)
    local maxUses = 3

    -- Draw uses as circles
    for i = 1, maxUses do
        local color = i <= uses and Color(0, 255, 0) or Color(50, 50, 50)
        draw.RoundedBox(8, 4 + (i - 1) * 12, h - 12, 8, 8, color)
    end
end
```

## Using Base Items

### Available Base Items

```lua
ITEM.base = "base_weapons"      -- Equippable weapons
ITEM.base = "base_ammo"         -- Ammunition
ITEM.base = "base_bags"         -- Containers
ITEM.base = "base_outfit"       -- Wearable clothing
ITEM.base = "base_pacoutfit"    -- PAC3 outfits
```

### Weapon Base Example

```lua
-- schema/items/sh_shotgun.lua
ITEM.name = "Shotgun"
ITEM.description = "A pump-action shotgun"
ITEM.model = "models/weapons/w_shotgun.mdl"
ITEM.base = "base_weapons"
ITEM.weapon = "weapon_shotgun"
ITEM.ammo = "buckshot"
ITEM.width = 3
ITEM.height = 1
ITEM.price = 800
ITEM.category = "Weapons"
```

### Outfit Base Example

```lua
-- schema/items/sh_police_uniform.lua
ITEM.name = "Police Uniform"
ITEM.description = "Standard police uniform"
ITEM.model = "models/props_c17/suitcase001a.mdl"
ITEM.base = "base_outfit"
ITEM.width = 2
ITEM.height = 2
ITEM.outfitCategory = "uniform"

ITEM.bodyGroups = {
    ["torso"] = 1,
    ["legs"] = 1
}
```

## Complete Item Examples

### Lockpick

```lua
-- schema/items/sh_lockpick.lua
ITEM.name = "Lockpick"
ITEM.description = "A tool for picking locks"
ITEM.model = "models/props_c17/tools_wrench01a.mdl"
ITEM.width = 1
ITEM.height = 1
ITEM.category = "Tools"
ITEM.price = 150

function ITEM:OnInstanced(invID, x, y, item)
    item:SetData("uses", 3)
end

function ITEM:GetDescription()
    return self.description .. "\n\nUses: " .. self:GetData("uses", 3)
end

ITEM.functions.Use = {
    name = "Pick Lock",
    OnRun = function(item)
        local client = item.player
        local trace = client:GetEyeTrace()
        local door = trace.Entity

        if not IsValid(door) or not door:IsDoor() then
            client:Notify("You must be looking at a door")
            return false
        end

        if door:GetPos():Distance(client:GetPos()) > 100 then
            client:Notify("Door is too far")
            return false
        end

        if not door:GetSaveTable().m_bLocked then
            client:Notify("Door is already unlocked")
            return false
        end

        -- Start lockpicking
        client:SetAction("Lockpicking...", 5, function()
            if math.random(1, 100) <= 75 then
                door:Fire("unlock")
                client:Notify("Lock picked successfully")

                -- Reduce uses
                local uses = item:GetData("uses", 3) - 1
                if uses <= 0 then
                    return true  -- Remove item
                else
                    item:SetData("uses", uses)
                end
            else
                client:Notify("Lockpick failed")
            end
        end)

        return false
    end
}
```

### Radio

```lua
-- schema/items/sh_radio.lua
ITEM.name = "Portable Radio"
ITEM.description = "A two-way radio for communication"
ITEM.model = "models/deadbodies/dead_male_civilian_radio.mdl"
ITEM.width = 1
ITEM.height = 2
ITEM.category = "Communication"
ITEM.price = 300

function ITEM:OnInstanced(invID, x, y, item)
    item:SetData("frequency", "100.0")
    item:SetData("enabled", false)
end

function ITEM:GetDescription()
    local desc = self.description
    local freq = self:GetData("frequency", "100.0")
    local enabled = self:GetData("enabled", false)

    desc = desc .. "\n\nFrequency: " .. freq .. " MHz"
    desc = desc .. "\nStatus: " .. (enabled and "On" or "Off")

    return desc
end

ITEM.functions.Toggle = {
    name = "Toggle Power",
    OnRun = function(item)
        local enabled = item:GetData("enabled", false)
        item:SetData("enabled", not enabled)

        item.player:EmitSound("buttons/button9.wav")
        item.player:Notify("Radio " .. (enabled and "disabled" or "enabled"))

        return false
    end
}

ITEM.functions.SetFreq = {
    name = "Set Frequency",
    OnRun = function(item)
        -- This would open a UI to set frequency
        -- For simplicity, just randomize
        local freq = math.random(80, 120) + math.random() * 0.9
        item:SetData("frequency", string.format("%.1f", freq))

        item.player:Notify("Frequency set to " .. item:GetData("frequency"))

        return false
    end
}
```

## Best Practices

### ✅ DO

- Place item files in `schema/items/` folder
- Use `sh_` prefix for item files
- Set all required properties (name, description, model)
- Use item:SetData() for instance-specific values
- Use appropriate base items when possible
- Validate input in OnRun functions
- Check IsValid() on entities
- Handle edge cases (nil values, disconnects)
- Return true to remove item, false to keep

### ❌ DON'T

- Don't create items outside `schema/items/`
- Don't modify ITEM table properties at runtime
- Don't use item properties for instance data
- Don't forget to check SERVER/CLIENT realm
- Don't trust client-provided data
- Don't create items without inventory space check
- Don't forget return value in OnRun
- Don't bypass item system

## Common Patterns

### Durability System

```lua
function ITEM:OnInstanced(invID, x, y, item)
    item:SetData("durability", 100)
end

function ITEM:GetDescription()
    local durability = self:GetData("durability", 100)
    return self.description .. "\n\nDurability: " .. durability .. "%"
end

function ITEM:DamageDurability(amount)
    local durability = self:GetData("durability", 100)
    durability = durability - amount

    if durability <= 0 then
        self:Remove()
    else
        self:SetData("durability", durability)
    end
end
```

### Faction-Restricted Items

```lua
ITEM.faction = FACTION_POLICE

function ITEM:CanTransfer(oldInv, newInv)
    local character = self:GetOwner():GetCharacter()

    if character:GetFaction() != self.faction then
        self:GetOwner():Notify("Only police can carry this")
        return false
    end

    return true
end
```

## See Also

- [Item System](../systems/items.md) - Detailed item system reference
- [Inventory System](../systems/inventory.md) - Inventory management
- [Custom Items Guide](../guides/custom-items.md) - Step-by-step guide
- Source: `gamemode/core/libs/sh_item.lua`
- Base Items: `gamemode/items/base/`
