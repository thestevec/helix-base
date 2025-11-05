# CAMI Permission System

> **Reference**: `gamemode/core/libs/thirdparty/sh_cami.lua`

CAMI (Common Admin Mod Interface) provides a standardized permission system that makes admin mods intercompatible and provides an abstract privilege interface for addons and plugins.

## ⚠️ Important: Use CAMI for Permissions

**Always use CAMI for permission checking** rather than hardcoding admin checks. The framework provides:
- Admin mod independence (works with any CAMI-compliant admin mod)
- Standardized privilege registration
- Usergroup inheritance system
- Async permission checking with callbacks
- Fallback permission handling
- Integration with all Helix systems
- Consistent permission API across plugins

## Core Concepts

### What is CAMI?

CAMI is a permission abstraction layer that allows your plugins and schema to work with any admin mod (ULX, ServerGuard, SAM, etc.) without modification. Instead of checking `IsAdmin()` or calling admin mod-specific functions, you register privileges with CAMI and check permissions through its standardized API.

**Benefits**:
- Works with any CAMI-compliant admin mod
- No admin mod-specific code
- Centralized permission management
- Flexible inheritance system
- Plugin permissions configurable by server owners

### Key Terms

- **Privilege**: A named permission that players can have (e.g., "Helix - Logs", "Helix - CharBan")
- **Usergroup**: A group of players with specific privileges (e.g., "admin", "moderator", "vip")
- **MinAccess**: Default minimum access level for a privilege ("user", "admin", "superadmin")
- **Inheritance**: Usergroups can inherit permissions from other groups
- **Actor**: The player performing an action
- **Target**: The player being acted upon (optional)

### Default Usergroups

**Reference**: `gamemode/core/libs/thirdparty/sh_cami.lua:53-69`

```lua
user         -- Base usergroup (everyone)
admin        -- Inherits from user
superadmin   -- Inherits from admin
```

## Using CAMI

### CAMI.RegisterPrivilege

**Reference**: `gamemode/core/libs/thirdparty/sh_cami.lua:177`

```lua
CAMI.RegisterPrivilege({
    Name = "Privilege Name",
    MinAccess = "admin",  -- "user", "admin", or "superadmin"
    Description = "Description of what this allows"
})
```

Registers a privilege that admin mods can assign to usergroups. Call this in plugin initialization.

**Complete Example**:
```lua
-- In plugin initialization
if (SERVER) then
    -- Register privilege for managing vendors
    CAMI.RegisterPrivilege({
        Name = "Helix - Manage Vendors",
        MinAccess = "admin",
        Description = "Allows creating and editing vendors"
    })

    -- Register privilege for viewing logs
    CAMI.RegisterPrivilege({
        Name = "Helix - View Logs",
        MinAccess = "admin",
        Description = "Allows viewing server logs in real-time"
    })

    -- Register privilege for character management
    CAMI.RegisterPrivilege({
        Name = "Helix - Character Admin",
        MinAccess = "admin",
        Description = "Allows editing and managing player characters"
    })

    -- User-level privilege
    CAMI.RegisterPrivilege({
        Name = "Helix - Use Radio",
        MinAccess = "user",
        Description = "Allows using the radio system"
    })

    -- Superadmin-only privilege
    CAMI.RegisterPrivilege({
        Name = "Helix - Server Management",
        MinAccess = "superadmin",
        Description = "Full server management capabilities"
    })
end
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't use simple IsAdmin checks for plugin features
if client:IsAdmin() then
    -- This bypasses permission system!
end

-- WRONG: Don't check admin mod-specific functions
if ULib and ULib.ucl.query(client, "ulx seeasay") then
    -- Not compatible with other admin mods!
end

-- WRONG: Don't forget to register privileges
-- If you don't register, admin mods can't configure it!
```

### CAMI.PlayerHasAccess

**Reference**: `gamemode/core/libs/thirdparty/sh_cami.lua:216-242`

```lua
CAMI.PlayerHasAccess(actor, privilegeName, callback, target, extraInfo)
```

Checks if a player has access to a privilege. Always asynchronous with callback.

**Parameters**:
- `actor` (Player): Player performing the action
- `privilegeName` (string): Name of the privilege
- `callback` (function): Called with `(hasAccess, reason)` when check completes
- `target` (Player, optional): Target player for the action
- `extraInfo` (table, optional): Additional information (e.g., `{Fallback = "user"}`)

**Complete Example**:
```lua
-- Check before allowing action
function PLUGIN:PlayerCanSeeVendor(client, vendor)
    CAMI.PlayerHasAccess(client, "Helix - Manage Vendors", function(hasAccess, reason)
        if hasAccess then
            -- Show vendor admin interface
            net.Start("OpenVendorAdmin")
                net.WriteEntity(vendor)
            net.Send(client)
        else
            client:Notify("You don't have permission to manage vendors")
        end
    end)
end

-- Command with CAMI check
ix.command.Add("GiveMoney", {
    description = "Give money to a player",
    arguments = {
        ix.type.player,
        ix.type.number
    },
    OnRun = function(self, client, target, amount)
        CAMI.PlayerHasAccess(client, "Helix - Give Money", function(hasAccess)
            if not hasAccess then
                return "@noPerm"
            end

            local character = target:GetCharacter()

            if character then
                character:GiveMoney(amount)
                target:Notify("You received " .. ix.currency.Get(amount))
                client:Notify("Gave " .. ix.currency.Get(amount) .. " to " .. target:Name())

                ix.log.Add(client, "giveMoney", target:Name(), amount)
            end
        end)
    end
})

-- Check with target player
function PLUGIN:CanKickPlayer(admin, target)
    CAMI.PlayerHasAccess(admin, "Helix - Kick", function(hasAccess, reason)
        if hasAccess then
            target:Kick("Kicked by " .. admin:Name())
            ix.log.Add(admin, "playerKick", target:Name())
        else
            admin:Notify("You don't have permission to kick players")
        end
    end, target)  -- Pass target as 4th argument
end

-- Check with fallback
function PLUGIN:CheckPermission(client, action)
    CAMI.PlayerHasAccess(client, "MyPlugin - " .. action, function(hasAccess)
        if hasAccess then
            self:PerformAction(client, action)
        end
    end, nil, {Fallback = "admin"})  -- Default to admin if not configured
end
```

### CAMI.GetPlayersWithAccess

**Reference**: `gamemode/core/libs/thirdparty/sh_cami.lua`

```lua
CAMI.GetPlayersWithAccess(privilegeName, callback, target, extraInfo)
```

Gets all players with access to a privilege. Useful for broadcasting to admins.

**Complete Example**:
```lua
-- Send log message to all admins with log access
function PLUGIN:BroadcastLog(message)
    CAMI.GetPlayersWithAccess("Helix - Logs", function(receivers)
        net.Start("ixLogStream")
            net.WriteString(message)
        net.Send(receivers)
    end)
end

-- Notify all moderators
function PLUGIN:NotifyModerators(message)
    CAMI.GetPlayersWithAccess("Helix - Moderate", function(receivers)
        for _, receiver in ipairs(receivers) do
            receiver:ChatPrint("[STAFF] " .. message)
        end
    end)
end

-- Show admin notification
function PLUGIN:AdminNotice(title, text)
    CAMI.GetPlayersWithAccess("Helix - Admin", function(receivers)
        for _, admin in ipairs(receivers) do
            admin:SendLua(string.format([[
                Derma_Message("%s", "%s")
            ]], text:gsub('"', '\\"'), title:gsub('"', '\\"')))
        end
    end)
end

-- Broadcast event to staff
function PLUGIN:OnSuspiciousActivity(client, reason)
    local message = string.format("%s triggered suspicious activity: %s",
        client:Name(),
        reason
    )

    CAMI.GetPlayersWithAccess("Helix - Logs", function(staffMembers)
        for _, staff in ipairs(staffMembers) do
            staff:ChatPrint("[ANTICHEAT] " .. message)
        end
    end)
end
```

## Command Integration

### Commands with CAMI Privileges

**Reference**: `gamemode/core/libs/sh_command.lua:195`

```lua
COMMAND.privilege = "Privilege Name"  -- Without "Helix - " prefix
```

Commands automatically register and check CAMI privileges.

**Complete Example**:
```lua
-- Command with automatic CAMI privilege
ix.command.Add("Ban", {
    description = "Ban a player from the server",
    privilege = "Ban",  -- Creates "Helix - Ban" privilege
    arguments = {
        ix.type.player,
        ix.type.number,
        ix.type.text
    },
    OnRun = function(self, client, target, duration, reason)
        -- Permission already checked by command system
        target:Ban(duration, reason)
        ix.log.Add(client, "playerBan", target:Name(), duration, reason)

        return string.format("Banned %s for %d minutes", target:Name(), duration)
    end
})

-- Superadmin-only command
ix.command.Add("ConfigSet", {
    description = "Set server configuration",
    superAdminOnly = true,  -- Creates privilege with MinAccess = "superadmin"
    arguments = {
        ix.type.string,
        ix.type.text
    },
    OnRun = function(self, client, key, value)
        ix.config.Set(key, value)
        return string.format("Set %s to %s", key, value)
    end
})

-- Admin-only command
ix.command.Add("CharSetModel", {
    description = "Set a character's model",
    adminOnly = true,  -- Creates privilege with MinAccess = "admin"
    arguments = {
        ix.type.character,
        ix.type.string
    },
    OnRun = function(self, client, character, model)
        character:SetModel(model)
        character:Save()

        local player = character:GetPlayer()
        if IsValid(player) then
            player:SetModel(model)
        end

        return "Model set successfully"
    end
})

-- Custom privilege check
ix.command.Add("SpecialAction", {
    description = "Perform special action",
    OnCheckAccess = function(self, client)
        -- Custom permission logic
        local character = client:GetCharacter()

        if not character then
            return false
        end

        -- Check both CAMI privilege and character flag
        CAMI.PlayerHasAccess(client, "Helix - Special Action", function(hasAccess)
            return hasAccess and character:HasFlags("s")
        end)
    end,
    OnRun = function(self, client)
        -- Perform action
    end
})
```

## Advanced Usage

### Custom Privilege Validation

```lua
CAMI.RegisterPrivilege({
    Name = "Helix - Edit Character",
    MinAccess = "admin",
    Description = "Allows editing characters",
    HasAccess = function(self, actor, target)
        -- Custom validation logic
        if not IsValid(target) then
            return true  -- Can always edit if no specific target
        end

        -- Can't edit characters of higher-ranked players
        if target:IsSuperAdmin() and not actor:IsSuperAdmin() then
            return false, "Cannot edit superadmin characters"
        end

        return true
    end
})
```

### Usergroup Management

```lua
-- Register custom usergroup
CAMI.RegisterUsergroup({
    Name = "moderator",
    Inherits = "admin"  -- Inherits all admin permissions
}, "MyPlugin")

-- Check usergroup inheritance
local inherits = CAMI.UsergroupInherits("moderator", "admin")  -- true

-- Get inheritance root
local root = CAMI.InheritanceRoot("moderator")  -- "admin"

-- Get all usergroups
local usergroups = CAMI.GetUsergroups()

for name, group in pairs(usergroups) do
    print(name, group.Inherits)
end
```

## Complete Example

### Plugin with Full CAMI Integration

```lua
PLUGIN.name = "Vendor System"
PLUGIN.description = "Manage NPC vendors"

if (SERVER) then
    -- Register privileges
    CAMI.RegisterPrivilege({
        Name = "Helix - Create Vendor",
        MinAccess = "admin",
        Description = "Allows creating new vendors"
    })

    CAMI.RegisterPrivilege({
        Name = "Helix - Edit Vendor",
        MinAccess = "admin",
        Description = "Allows editing existing vendors"
    })

    CAMI.RegisterPrivilege({
        Name = "Helix - Delete Vendor",
        MinAccess = "superadmin",
        Description = "Allows deleting vendors"
    })

    -- Command with CAMI check
    ix.command.Add("VendorCreate", {
        description = "Create a vendor at your position",
        privilege = "Create Vendor",
        arguments = {
            ix.type.string  -- Vendor name
        },
        OnRun = function(self, client, name)
            local trace = client:GetEyeTrace()
            local vendor = self:CreateVendor(trace.HitPos, name)

            ix.log.Add(client, "vendorCreate", name)
            return "Created vendor: " .. name
        end
    })

    ix.command.Add("VendorDelete", {
        description = "Delete the vendor you're looking at",
        privilege = "Delete Vendor",
        OnRun = function(self, client)
            local trace = client:GetEyeTrace()
            local entity = trace.Entity

            if not IsValid(entity) or entity:GetClass() != "ix_vendor" then
                return "Not looking at a vendor"
            end

            local name = entity:GetNetVar("name", "Unknown")
            entity:Remove()

            ix.log.Add(client, "vendorDelete", name)
            return "Deleted vendor: " .. name
        end
    })

    -- Use CAMI in entity interaction
    function PLUGIN:PlayerUse(client, entity)
        if entity:GetClass() == "ix_vendor" then
            -- Check if player wants to edit (holding shift)
            if client:KeyDown(IN_SPEED) then
                CAMI.PlayerHasAccess(client, "Helix - Edit Vendor", function(hasAccess)
                    if hasAccess then
                        -- Open edit menu
                        net.Start("VendorEdit")
                            net.WriteEntity(entity)
                        net.Send(client)
                    else
                        -- Open shop menu
                        net.Start("VendorShop")
                            net.WriteEntity(entity)
                        net.Send(client)
                    end
                end)

                return false
            end
        end
    end

    -- Broadcast vendor changes to admins
    function PLUGIN:OnVendorModified(client, vendor, changes)
        local message = string.format("%s modified vendor '%s'",
            client:Name(),
            vendor:GetNetVar("name")
        )

        CAMI.GetPlayersWithAccess("Helix - Edit Vendor", function(admins)
            for _, admin in ipairs(admins) do
                if admin != client then
                    admin:Notify(message)
                end
            end
        end)
    end
end
```

## Best Practices

### ✅ DO

- Register all plugin privileges with descriptive names
- Use `CAMI.PlayerHasAccess()` for permission checks
- Always use callbacks (CAMI is asynchronous)
- Prefix privileges with "Helix - " for consistency
- Set appropriate `MinAccess` levels
- Provide clear privilege descriptions
- Use `CAMI.GetPlayersWithAccess()` for admin broadcasts
- Let commands auto-register privileges with `privilege` field

### ❌ DON'T

- Don't use `IsAdmin()` or `IsSuperAdmin()` for plugin features
- Don't check admin mod-specific functions (ULX, ServerGuard, etc.)
- Don't forget callbacks are asynchronous
- Don't bypass CAMI for "convenience"
- Don't forget to register privileges
- Don't assume privilege checks are instant
- Don't hardcode permission levels
- Don't create custom permission systems

## Common Patterns

### Pattern 1: Command with Privilege

```lua
ix.command.Add("MyCommand", {
    privilege = "My Action",  -- Auto-creates "Helix - My Action"
    arguments = {ix.type.player},
    OnRun = function(self, client, target)
        -- Permission already checked
    end
})
```

### Pattern 2: Manual Permission Check

```lua
function PLUGIN:SomeAction(client)
    CAMI.PlayerHasAccess(client, "Helix - My Privilege", function(hasAccess)
        if hasAccess then
            -- Perform action
        else
            client:Notify("No permission")
        end
    end)
end
```

### Pattern 3: Broadcast to Admins

```lua
function PLUGIN:NotifyAdmins(message)
    CAMI.GetPlayersWithAccess("Helix - Logs", function(admins)
        for _, admin in ipairs(admins) do
            admin:ChatPrint("[ADMIN] " .. message)
        end
    end)
end
```

### Pattern 4: Privilege with Custom Validation

```lua
CAMI.RegisterPrivilege({
    Name = "Helix - My Privilege",
    MinAccess = "admin",
    HasAccess = function(self, actor, target)
        -- Custom logic
        return someCondition
    end
})
```

## Common Issues

### Issue: Permission Check Not Working

**Cause**: Forgetting asynchronous callback
**Fix**: Always use callback function

```lua
-- WRONG
local hasAccess = CAMI.PlayerHasAccess(client, "Helix - Admin")  -- No return value!

-- CORRECT
CAMI.PlayerHasAccess(client, "Helix - Admin", function(hasAccess)
    if hasAccess then
        -- Do something
    end
end)
```

### Issue: Privilege Not Showing in Admin Mod

**Cause**: Privilege not registered
**Fix**: Register privilege on server init

```lua
if (SERVER) then
    CAMI.RegisterPrivilege({
        Name = "Helix - My Feature",
        MinAccess = "admin"
    })
end
```

## See Also

- [Commands System](../systems/commands.md) - Command privilege integration
- [Logging System](./logging.md) - Admin log access
- [Configuration System](../systems/configuration.md) - Config access
- CAMI Specification: https://github.com/glua/CAMI
- Source: `gamemode/core/libs/thirdparty/sh_cami.lua`
