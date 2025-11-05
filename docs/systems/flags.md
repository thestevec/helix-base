# Flag System

> **Reference**: `gamemode/core/libs/sh_flag.lua`, `gamemode/core/meta/sh_character.lua`

Flags are single-character permissions that control what characters can do.

## ⚠️ Important: Use Built-in Flag System

**Always use character flag methods**. Don't create custom permission systems. Framework provides:
- Database persistence
- Network synchronization
- Easy flag checking
- Integration with commands/items

## Common Flags

```
"1" - Default flag (all characters)
"a" - Access restricted areas
"m" - Map editing (context menu)
"p" - Physgun
"t" - Tool gun
"v" - Vendor access
"b" - Business ownership
"d" - Door access
```

## Using Flags

### Giving Flags

```lua
-- SERVER ONLY
character:GiveFlags("v")    -- Give vendor flag
character:GiveFlags("vpm")  -- Give multiple flags
```

### Removing Flags

```lua
-- SERVER ONLY
character:TakeFlags("v")
```

### Checking Flags

```lua
-- SHARED
if character:HasFlags("v") then
    -- Character has vendor flag
end

-- Check multiple (has ANY)
if character:HasFlags("vpm") then
    -- Has v OR p OR m
end
```

### Getting All Flags

```lua
local flags = character:GetFlags()  -- Returns "vpm1a"
```

## Flag Examples

### Item with Flag Requirement

```lua
ITEM.flag = "v"  -- Requires vendor flag to use

function ITEM:CanTransfer(oldInv, newInv)
    local character = self:GetOwner():GetCharacter()

    if self.flag and not character:HasFlags(self.flag) then
        return false
    end
    return true
end
```

### Command with Flag

```lua
ix.command.Add("VendorMenu", {
    OnCheckAccess = function(self, client)
        local char = client:GetCharacter()
        return char and char:HasFlags("v")
    end,
    OnRun = function(self, client)
        -- Open vendor menu
    end
})
```

### Faction with Flag Requirement

```lua
FACTION.flag = "p"  -- Need 'p' flag to create character in this faction
```

## Best Practices

### ✅ DO

- Use single characters for flags ("v", "p", "m")
- Use lowercase letters
- Check flags before restricted actions
- Give flags via admin commands
- Use HasFlags() for checks

### ❌ DON'T

- Don't use multi-character flags ("vendor", "police")
- Don't modify character.vars directly
- Don't create custom permission systems

## See Also

- [Character System](character.md)
- Flag System: `gamemode/core/libs/sh_flag.lua`
