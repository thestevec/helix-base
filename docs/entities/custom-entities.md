# Creating Custom Entities

This guide covers creating custom entities for the Helix framework, including interactive entities, persistent entities, and entities that integrate with Helix systems.

## ⚠️ Important: Follow Helix Patterns

**Always follow Helix entity patterns** rather than creating standalone GMod entities. The framework provides:
- Network variable system (`SetNetVar`, `GetNetVar`)
- Interaction system with progress bars
- Tooltip/entity info system
- Persistence options
- Integration with character/inventory systems

## Core Concepts

### Entity Structure

Helix entities follow standard GMod entity structure but integrate with framework features:

```
entities/
└── entities/
    └── my_custom_entity.lua
```

OR in a plugin:

```
plugins/
└── myplugin/
    └── entities/
        └── entities/
            └── my_custom_entity.lua
```

### Essential Entity Properties

```lua
AddCSLuaFile()  -- Always include for client download

ENT.Type = "anim"              -- Entity type
ENT.PrintName = "My Entity"    -- Display name in spawn menu
ENT.Category = "Helix"         -- Spawn menu category
ENT.Spawnable = true           -- Can be spawned from menu
ENT.AdminOnly = false          -- Requires admin to spawn
ENT.bNoPersist = true          -- Don't save entity on map cleanup
ENT.ShowPlayerInteraction = true  -- Show interaction tooltip
```

## Basic Custom Entity

### Simple Interactive Entity

```lua
AddCSLuaFile()

ENT.Type = "anim"
ENT.PrintName = "Storage Box"
ENT.Category = "Helix"
ENT.Spawnable = true
ENT.AdminOnly = false
ENT.bNoPersist = true
ENT.ShowPlayerInteraction = true

-- Network variables
function ENT:SetupDataTables()
    self:NetworkVar("String", 0, "OwnerName")
    self:NetworkVar("Int", 0, "ItemCount")
end

if SERVER then
    function ENT:Initialize()
        self:SetModel("models/props_c17/FurnitureDrawer001a.mdl")
        self:SetSolid(SOLID_VPHYSICS)
        self:PhysicsInit(SOLID_VPHYSICS)
        self:SetUseType(SIMPLE_USE)

        local phys = self:GetPhysicsObject()

        if IsValid(phys) then
            phys:EnableMotion(true)
            phys:Wake()
        end

        -- Initialize data
        self:SetOwnerName("Unknown")
        self:SetItemCount(0)
    end

    function ENT:Use(activator)
        if not activator:IsPlayer() then
            return
        end

        -- Use Helix interaction system
        activator:PerformInteraction(1, self, function(client)
            client:Notify("You opened the storage box!")
            -- Open custom UI, add items, etc.
        end)
    end
else
    -- Client-side rendering
    function ENT:Draw()
        self:DrawModel()
    end

    -- Tooltip system
    ENT.PopulateEntityInfo = true

    function ENT:OnPopulateEntityInfo(container)
        local name = container:AddRow("name")
        name:SetImportant()
        name:SetText("Storage Box")
        name:SizeToContents()

        local owner = container:AddRow("owner")
        owner:SetText("Owner: " .. self:GetOwnerName())
        owner:SizeToContents()

        local count = container:AddRow("count")
        count:SetText("Items: " .. self:GetItemCount())
        count:SizeToContents()
    end
end
```

## Using Helix Features

### Network Variables (NetVars)

Helix provides an enhanced network variable system:

```lua
-- Set network variable (server-side)
self:SetNetVar("key", value, receiver)

-- Get network variable (both sides)
local value = self:GetNetVar("key", default)
```

**Example**:

```lua
if SERVER then
    function ENT:Initialize()
        -- Set various types
        self:SetNetVar("owner", 0)
        self:SetNetVar("locked", false)
        self:SetNetVar("title", "My Entity")
        self:SetNetVar("data", {key = "value"})
    end

    function ENT:LockEntity(client)
        self:SetNetVar("locked", true)
        self:SetNetVar("owner", client:GetCharacter():GetID())
    end
end

-- Access from either realm
if self:GetNetVar("locked", false) then
    -- Entity is locked
end
```

### Interaction System

**Reference**: See `ix_item.lua:46` and `ix_money.lua:45`

```lua
-- Show progress bar during interaction
player:PerformInteraction(duration, entity, callback)
```

**Example**:

```lua
function ENT:Use(activator)
    local character = activator:GetCharacter()

    if not character then
        return
    end

    -- Check if already being used
    if self.ixInteractionDirty then
        return
    end

    -- 2 second interaction
    activator:PerformInteraction(2, self, function(client)
        -- Called when interaction completes
        self:OnInteractionComplete(client)

        -- Return true to mark dirty (prevent immediate reuse)
        return true
    end)
end

function ENT:OnInteractionComplete(client)
    client:Notify("Interaction complete!")
    -- Do something
end
```

### Entity Info/Tooltip

**Reference**: See `ix_item.lua:186` and `ix_vendor.lua:331`

```lua
ENT.PopulateEntityInfo = true

function ENT:OnPopulateEntityInfo(container)
    -- Add rows to tooltip
    local row = container:AddRow("identifier")
    row:SetImportant()  -- Makes it bold/colored
    row:SetText("Text to display")
    row:SetBackgroundColor(Color(255, 0, 0))  -- Optional
    row:SizeToContents()
end
```

**Example**:

```lua
function ENT:OnPopulateEntityInfo(container)
    -- Title
    local name = container:AddRow("name")
    name:SetImportant()
    name:SetText(self:GetNetVar("title", "Unnamed"))
    name:SizeToContents()

    -- Description
    local desc = container:AddRow("description")
    desc:SetText("This is a custom entity")
    desc:SizeToContents()

    -- Owner (if owned)
    local ownerID = self:GetNetVar("owner", 0)

    if ownerID > 0 then
        local owner = ix.char.loaded[ownerID]

        if owner then
            local ownerRow = container:AddRow("owner")
            ownerRow:SetText("Owner: " .. owner:GetName())
            ownerRow:SizeToContents()
        end
    end

    -- Status with color
    local locked = self:GetNetVar("locked", false)
    local status = container:AddRow("status")
    status:SetText(locked and "Locked" or "Unlocked")
    status:SetBackgroundColor(locked and Color(200, 50, 50) or Color(50, 200, 50))
    status:SizeToContents()
end
```

### Entity Menu (Right-Click)

```lua
function ENT:GetEntityMenu(client)
    local options = {}

    -- Add menu options
    options["Examine"] = function()
        -- Called when clicked
        client:Notify("You examine the entity")
        return false  -- Return false to prevent network send
    end

    options["Lock"] = function()
        -- Send to server
        net.Start("MyEntityLock")
            net.WriteEntity(self)
        net.SendToServer()

        return false  -- We're handling networking manually
    end

    return options
end
```

## Advanced Examples

### Storage Container Entity

```lua
AddCSLuaFile()

ENT.Type = "anim"
ENT.PrintName = "Storage Container"
ENT.Category = "Helix"
ENT.Spawnable = true
ENT.bNoPersist = true
ENT.ShowPlayerInteraction = true

function ENT:SetupDataTables()
    self:NetworkVar("Int", 0, "InventoryID")
end

if SERVER then
    function ENT:Initialize()
        self:SetModel("models/props_c17/FurnitureDrawer001a.mdl")
        self:SetSolid(SOLID_VPHYSICS)
        self:PhysicsInit(SOLID_VPHYSICS)
        self:SetUseType(SIMPLE_USE)

        local phys = self:GetPhysicsObject()

        if IsValid(phys) then
            phys:EnableMotion(false)
            phys:Sleep()
        end

        -- Create inventory for this container
        ix.inventory.New(0, "storage6x4", function(inventory)
            if not inventory then
                self:Remove()
                return
            end

            self.ixInventory = inventory
            self:SetInventoryID(inventory:GetID())
        end)
    end

    function ENT:Use(activator)
        local character = activator:GetCharacter()

        if not character or not self.ixInventory then
            return
        end

        activator:PerformInteraction(1, self, function(client)
            -- Open the inventory
            self.ixInventory:Sync(client)

            net.Start("ixInventoryOpen")
                net.WriteUInt(self.ixInventory:GetID(), 32)
            net.Send(client)
        end)
    end

    function ENT:OnRemove()
        if self.ixInventory then
            self.ixInventory:Remove()
        end
    end
else
    function ENT:Draw()
        self:DrawModel()
    end

    ENT.PopulateEntityInfo = true

    function ENT:OnPopulateEntityInfo(container)
        local name = container:AddRow("name")
        name:SetImportant()
        name:SetText("Storage Container")
        name:SizeToContents()

        local desc = container:AddRow("description")
        desc:SetText("Press E to open")
        desc:SizeToContents()
    end
end
```

### Timed Entity with Effects

```lua
AddCSLuaFile()

ENT.Type = "anim"
ENT.PrintName = "Time Bomb"
ENT.Category = "Helix"
ENT.Spawnable = true
ENT.AdminOnly = true
ENT.bNoPersist = true

function ENT:SetupDataTables()
    self:NetworkVar("Int", 0, "ExplodeTime")
    self:NetworkVar("Bool", 0, "Armed")
end

if SERVER then
    function ENT:Initialize()
        self:SetModel("models/props_c17/BriefCase001a.mdl")
        self:SetSolid(SOLID_VPHYSICS)
        self:PhysicsInit(SOLID_VPHYSICS)
        self:SetUseType(SIMPLE_USE)

        local phys = self:GetPhysicsObject()

        if IsValid(phys) then
            phys:EnableMotion(true)
            phys:Wake()
        end

        self:SetArmed(false)
        self:SetExplodeTime(0)
    end

    function ENT:Use(activator)
        if self:GetArmed() then
            activator:Notify("It's already armed!")
            return
        end

        activator:PerformInteraction(3, self, function(client)
            self:ArmBomb(client)
        end)
    end

    function ENT:ArmBomb(client)
        self:SetArmed(true)
        self:SetExplodeTime(CurTime() + 30)

        self:EmitSound("buttons/button14.wav")

        timer.Simple(30, function()
            if IsValid(self) then
                self:Explode()
            end
        end)

        ix.log.Add(client, "bombArm", self:GetPos())
    end

    function ENT:Explode()
        local pos = self:GetPos()

        -- Create explosion
        local explode = ents.Create("env_explosion")
        explode:SetPos(pos)
        explode:SetKeyValue("iMagnitude", "200")
        explode:Spawn()
        explode:Fire("Explode")

        -- Damage nearby entities
        for _, ent in ipairs(ents.FindInSphere(pos, 300)) do
            if ent:IsPlayer() then
                ent:TakeDamage(100, self, self)
            end
        end

        self:Remove()
    end

    function ENT:Think()
        if self:GetArmed() then
            local timeLeft = self:GetExplodeTime() - CurTime()

            if timeLeft <= 10 and timeLeft > 0 then
                -- Beep faster as time runs out
                if not self.nextBeep or self.nextBeep < CurTime() then
                    self:EmitSound("buttons/button17.wav")
                    self.nextBeep = CurTime() + (timeLeft / 10)
                end
            end
        end

        self:NextThink(CurTime() + 0.1)
        return true
    end
else
    function ENT:Draw()
        self:DrawModel()

        -- Draw timer if armed
        if self:GetArmed() then
            local pos = self:GetPos() + self:GetUp() * 10
            local ang = self:GetAngles()

            local timeLeft = math.max(0, self:GetExplodeTime() - CurTime())

            ang:RotateAroundAxis(ang:Up(), 90)
            ang:RotateAroundAxis(ang:Forward(), 90)

            cam.Start3D2D(pos, ang, 0.1)
                draw.SimpleText(
                    string.format("%d", math.ceil(timeLeft)),
                    "DermaLarge",
                    0, 0,
                    self:GetArmed() and Color(255, 0, 0) or Color(255, 255, 255),
                    TEXT_ALIGN_CENTER,
                    TEXT_ALIGN_CENTER
                )
            cam.End3D2D()
        end
    end

    ENT.PopulateEntityInfo = true

    function ENT:OnPopulateEntityInfo(container)
        local name = container:AddRow("name")
        name:SetImportant()
        name:SetText("Time Bomb")
        name:SetBackgroundColor(Color(200, 50, 50))
        name:SizeToContents()

        if self:GetArmed() then
            local time = container:AddRow("time")
            time:SetText(string.format("Explodes in: %ds", math.ceil(self:GetExplodeTime() - CurTime())))
            time:SizeToContents()
        else
            local desc = container:AddRow("description")
            desc:SetText("Press E to arm (3s)")
            desc:SizeToContents()
        end
    end
end
```

### Entity with Custom Menu

```lua
if SERVER then
    util.AddNetworkString("MyEntityAction")

    net.Receive("MyEntityAction", function(length, client)
        local entity = net.ReadEntity()
        local action = net.ReadString()

        if not IsValid(entity) or entity:GetClass() != "my_custom_entity" then
            return
        end

        if action == "lock" then
            entity:SetNetVar("locked", true)
            client:Notify("Locked!")
        elseif action == "unlock" then
            entity:SetNetVar("locked", false)
            client:Notify("Unlocked!")
        end
    end)
end

function ENT:GetEntityMenu(client)
    local options = {}
    local locked = self:GetNetVar("locked", false)

    if locked then
        options["Unlock"] = function()
            net.Start("MyEntityAction")
                net.WriteEntity(self)
                net.WriteString("unlock")
            net.SendToServer()

            return false
        end
    else
        options["Lock"] = function()
            net.Start("MyEntityAction")
                net.WriteEntity(self)
                net.WriteString("lock")
            net.SendToServer()

            return false
        end
    end

    options["Examine"] = function()
        local panel = vgui.Create("DFrame")
        panel:SetSize(400, 300)
        panel:Center()
        panel:SetTitle("Entity Info")
        panel:MakePopup()

        local label = panel:Add("DLabel")
        label:SetPos(10, 30)
        label:SetText("This is my custom entity!")
        label:SizeToContents()

        return false
    end

    return options
end
```

## Best Practices

### ✅ DO

- Use `AddCSLuaFile()` at the top of every entity file
- Use Helix's `SetNetVar`/`GetNetVar` for network synchronization
- Use `PerformInteraction()` for timed interactions
- Implement `PopulateEntityInfo` for tooltips
- Check validity with `IsValid()` before using entities
- Use `bNoPersist = true` if entity shouldn't save
- Clean up resources in `OnRemove()`
- Use appropriate `SetUseType()` (SIMPLE_USE, CONTINUOUS_USE)

### ❌ DON'T

- Don't forget `AddCSLuaFile()` (clients won't download)
- Don't use standard GMod `SetNW*` functions (use SetNetVar)
- Don't create entities without proper initialization
- Don't forget to validate players have characters
- Don't create entities that persist without cleanup
- Don't forget realm checks (`if SERVER then` / `if CLIENT then`)
- Don't bypass Helix's interaction system

## Common Patterns

### Owned Entities

```lua
function ENT:SetOwner(character)
    self:SetNetVar("owner", character:GetID())
    self:SetNetVar("ownerName", character:GetName())
end

function ENT:IsOwner(client)
    local character = client:GetCharacter()

    if not character then
        return false
    end

    return self:GetNetVar("owner", 0) == character:GetID()
end

function ENT:Use(activator)
    if not self:IsOwner(activator) then
        activator:Notify("This doesn't belong to you!")
        return
    end

    -- Continue with use logic
end
```

### Money Cost Interaction

```lua
function ENT:Use(activator)
    local character = activator:GetCharacter()

    if not character then
        return
    end

    local cost = 500

    if not character:HasMoney(cost) then
        activator:Notify("You need " .. ix.currency.Get(cost))
        return
    end

    activator:PerformInteraction(2, self, function(client)
        local char = client:GetCharacter()

        if char and char:HasMoney(cost) then
            char:TakeMoney(cost)
            self:OnPurchase(client)
        end
    end)
end
```

### Save/Load Entity Data

```lua
-- In plugin
function PLUGIN:SaveData()
    local data = {}

    for _, ent in ipairs(ents.FindByClass("my_custom_entity")) do
        data[#data + 1] = {
            pos = ent:GetPos(),
            angles = ent:GetAngles(),
            model = ent:GetModel(),
            owner = ent:GetNetVar("owner", 0)
        }
    end

    self:SetData(data)
end

function PLUGIN:LoadData()
    for _, data in ipairs(self:GetData() or {}) do
        local ent = ents.Create("my_custom_entity")
        ent:SetPos(data.pos)
        ent:SetAngles(data.angles)
        ent:SetModel(data.model)
        ent:Spawn()
        ent:SetNetVar("owner", data.owner)
    end
end
```

## Common Issues

### Entity Not Showing for Clients

**Cause**: Missing `AddCSLuaFile()`
**Fix**: Add at the top of entity file

```lua
AddCSLuaFile()  -- Must be first line
```

### Network Variables Not Syncing

**Cause**: Using wrong network system
**Fix**: Use Helix's NetVar system

```lua
-- WRONG
self:SetNWString("key", "value")

-- CORRECT
self:SetNetVar("key", "value")
```

### Entity Falls Through Floor

**Cause**: Physics not initialized or spawned at wrong position
**Fix**: Proper physics initialization and positioning

```lua
function ENT:Initialize()
    self:SetModel("...")
    self:SetSolid(SOLID_VPHYSICS)
    self:PhysicsInit(SOLID_VPHYSICS)  -- Initialize physics

    local phys = self:GetPhysicsObject()

    if IsValid(phys) then
        phys:Wake()
        phys:EnableMotion(false)  -- For static entities
    end
end
```

### Interaction Doesn't Work

**Cause**: Wrong use type or missing interaction callback
**Fix**: Set proper use type and validate callback

```lua
function ENT:Initialize()
    self:SetUseType(SIMPLE_USE)  -- or CONTINUOUS_USE
end

function ENT:Use(activator)
    activator:PerformInteraction(1, self, function(client)
        -- Make sure function exists and is called
        print("Interaction complete!")
    end)
end
```

## See Also

- [ix_item Entity](ix_item.md) - Item entity implementation reference
- [ix_vendor Entity](ix_vendor.md) - Complex entity example
- [Inventory System](../systems/inventory.md) - For storage entities
- [Character System](../systems/character.md) - For ownership
- [Networking Library](../libraries/networking.md) - Network communication (if documented)
- GMod Wiki: [Scripting Entities](https://wiki.facepunch.com/gmod/Scripting_Entities)
