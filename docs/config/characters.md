# Character Configuration

> **Reference**: `gamemode/config/sh_config.lua:14-144`

Configuration options for character creation, limits, movement, persistence, and player attributes.

## ⚠️ Important: Use Built-in Config System

**Always use `ix.config.Get()` to retrieve character settings** rather than hardcoding values. The framework provides:
- Centralized character settings management
- Admin-configurable values via F1 menu
- Automatic validation and constraints
- Database persistence
- Real-time updates with callbacks

## Character Limits

### maxCharacters

**Reference**: `gamemode/config/sh_config.lua:14`

```lua
ix.config.Add("maxCharacters", 5, "The maximum number of characters a player can have.", nil, {
    data = {min = 1, max = 50},
    category = "characters"
})
```

**Type**: Number (Integer)
**Default**: `5`
**Range**: 1 - 50
**Description**: Maximum number of characters each player can create

**Usage**:
```lua
-- Check if player can create more characters
local current = #client:GetCharacters()
local max = ix.config.Get("maxCharacters", 5)

if current >= max then
    client:Notify("You have reached the maximum number of characters!")
    return false
end
```

---

### maxAttributes

**Reference**: `gamemode/config/sh_config.lua:40`

```lua
ix.config.Add("maxAttributes", 100, "The maximum amount each attribute can be.", nil, {
    data = {min = 0, max = 100},
    category = "characters"
})
```

**Type**: Number (Integer)
**Default**: `100`
**Range**: 0 - 100
**Description**: Maximum value for each character attribute (strength, endurance, etc.)

**Usage**:
```lua
-- Ensure attribute doesn't exceed maximum
local current = character:GetAttribute("strength", 0)
local max = ix.config.Get("maxAttributes", 100)

if current >= max then
    client:Notify("Attribute is at maximum!")
    return
end
```

---

## Character Creation

### minNameLength

**Reference**: `gamemode/config/sh_config.lua:82`

```lua
ix.config.Add("minNameLength", 4, "The minimum number of characters in a name.", nil, {
    data = {min = 4, max = 64},
    category = "characters"
})
```

**Type**: Number (Integer)
**Default**: `4`
**Range**: 4 - 64
**Description**: Minimum number of characters required for character names

---

### maxNameLength

**Reference**: `gamemode/config/sh_config.lua:86`

```lua
ix.config.Add("maxNameLength", 32, "The maximum number of characters in a name.", nil, {
    data = {min = 16, max = 128},
    category = "characters"
})
```

**Type**: Number (Integer)
**Default**: `32`
**Range**: 16 - 128
**Description**: Maximum number of characters allowed for character names

**Usage**:
```lua
-- Validate character name
local name = "John Doe"
local minLen = ix.config.Get("minNameLength", 4)
local maxLen = ix.config.Get("maxNameLength", 32)

if #name < minLen then
    return false, "Name is too short (minimum " .. minLen .. " characters)"
elseif #name > maxLen then
    return false, "Name is too long (maximum " .. maxLen .. " characters)"
end
```

---

### minDescriptionLength

**Reference**: `gamemode/config/sh_config.lua:90`

```lua
ix.config.Add("minDescriptionLength", 16, "The minimum number of characters in a description.", nil, {
    data = {min = 0, max = 300},
    category = "characters"
})
```

**Type**: Number (Integer)
**Default**: `16`
**Range**: 0 - 300
**Description**: Minimum number of characters required for character descriptions

**Usage**:
```lua
-- Validate description
local desc = character:GetDescription()
local minLen = ix.config.Get("minDescriptionLength", 16)

if #desc < minLen then
    client:Notify("Description must be at least " .. minLen .. " characters")
    return false
end
```

---

## Character Persistence

### saveInterval

**Reference**: `gamemode/config/sh_config.lua:94`

```lua
ix.config.Add("saveInterval", 300, "How often characters save in seconds.", nil, {
    data = {min = 60, max = 3600},
    category = "characters"
})
```

**Type**: Number (Integer)
**Default**: `300` (5 minutes)
**Range**: 60 - 3600 seconds
**Description**: How frequently character data is automatically saved to database

**Usage**:
```lua
-- The framework automatically saves characters at this interval
-- You can also manually save:
character:Save()
```

---

### spawnTime

**Reference**: `gamemode/config/sh_config.lua:70`

```lua
ix.config.Add("spawnTime", 5, "The time it takes to respawn.", nil, {
    data = {min = 0, max = 10000},
    category = "characters"
})
```

**Type**: Number (Integer)
**Default**: `5`
**Range**: 0 - 10000 seconds
**Description**: Time in seconds before a player respawns after death

---

## Character Movement

### walkSpeed

**Reference**: `gamemode/config/sh_config.lua:98`

```lua
ix.config.Add("walkSpeed", 130, "How fast a player normally walks.", function(oldValue, newValue)
    for _, v in player.Iterator() do
        v:SetWalkSpeed(newValue)
    end
end, {
    data = {min = 75, max = 500},
    category = "characters"
})
```

**Type**: Number (Integer)
**Default**: `130`
**Range**: 75 - 500 units/second
**Description**: Base walking speed for all characters

**Callback**: Automatically updates all connected players' walk speeds when changed

---

### runSpeed

**Reference**: `gamemode/config/sh_config.lua:106`

```lua
ix.config.Add("runSpeed", 235, "How fast a player normally runs.", function(oldValue, newValue)
    for _, v in player.Iterator() do
        v:SetRunSpeed(newValue)
    end
end, {
    data = {min = 75, max = 500},
    category = "characters"
})
```

**Type**: Number (Integer)
**Default**: `235`
**Range**: 75 - 500 units/second
**Description**: Base running speed for all characters

**Callback**: Automatically updates all connected players' run speeds when changed

---

### walkRatio

**Reference**: `gamemode/config/sh_config.lua:114`

```lua
ix.config.Add("walkRatio", 0.5, "How fast one goes when holding ALT.", nil, {
    data = {min = 0, max = 1, decimals = 1},
    category = "characters"
})
```

**Type**: Number (Float)
**Default**: `0.5` (50% speed)
**Range**: 0.0 - 1.0
**Description**: Speed multiplier when holding the walk key (ALT by default)

**Usage**:
```lua
-- Calculate slow walk speed
local walkSpeed = ix.config.Get("walkSpeed", 130)
local walkRatio = ix.config.Get("walkRatio", 0.5)
local slowWalkSpeed = walkSpeed * walkRatio
```

---

## Inventory Settings

### inventoryWidth

**Reference**: `gamemode/config/sh_config.lua:74`

```lua
ix.config.Add("inventoryWidth", 6, "How many slots in a row there is in a default inventory.", nil, {
    data = {min = 0, max = 20},
    category = "characters"
})
```

**Type**: Number (Integer)
**Default**: `6`
**Range**: 0 - 20
**Description**: Number of horizontal slots in default character inventories

---

### inventoryHeight

**Reference**: `gamemode/config/sh_config.lua:78`

```lua
ix.config.Add("inventoryHeight", 4, "How many slots in a column there is in a default inventory.", nil, {
    data = {min = 0, max = 20},
    category = "characters"
})
```

**Type**: Number (Integer)
**Default**: `4`
**Range**: 0 - 20
**Description**: Number of vertical slots in default character inventories

**Usage**:
```lua
-- Create inventory with config dimensions
local width = ix.config.Get("inventoryWidth", 6)
local height = ix.config.Get("inventoryHeight", 4)

ix.inventory.Create(width, height, 1, function(inventory)
    character:SetInventory(inventory)
end)
```

---

## Economy

### defaultMoney

**Reference**: `gamemode/config/sh_config.lua:137`

```lua
ix.config.Add("defaultMoney", 0, "The amount of money that players start with.", nil, {
    category = "characters",
    data = {min = 0, max = 1000}
})
```

**Type**: Number (Integer)
**Default**: `0`
**Range**: 0 - 1000
**Description**: Starting money for new characters

**Usage**:
```lua
-- Give new character starting money
function PLUGIN:OnCharacterCreated(client, character)
    local startMoney = ix.config.Get("defaultMoney", 0)
    character:SetMoney(startMoney)
end
```

---

### minMoneyDropAmount

**Reference**: `gamemode/config/sh_config.lua:141`

```lua
ix.config.Add("minMoneyDropAmount", 1, "The minimum amount of money that can be dropped.", nil, {
    category = "characters",
    data = {min = 1, max = 1000}
})
```

**Type**: Number (Integer)
**Default**: `1`
**Range**: 1 - 1000
**Description**: Minimum amount of money that can be dropped as a world entity

---

## Other Settings

### scoreboardRecognition

**Reference**: `gamemode/config/sh_config.lua:134`

```lua
ix.config.Add("scoreboardRecognition", false, "Whether or not recognition is used in the scoreboard.", nil, {
    category = "characters"
})
```

**Type**: Boolean
**Default**: `false`
**Description**: When enabled, the scoreboard shows recognition status instead of actual names

---

## Complete Example: Using Character Config

```lua
-- Validate character creation
function SCHEMA:CanPlayerCreateCharacter(client, payload)
    -- Check character limit
    local current = #client:GetCharacters()
    local max = ix.config.Get("maxCharacters", 5)

    if current >= max then
        return false, "You have reached the maximum number of characters (" .. max .. ")"
    end

    -- Validate name length
    local name = payload.name or ""
    local minName = ix.config.Get("minNameLength", 4)
    local maxName = ix.config.Get("maxNameLength", 32)

    if #name < minName then
        return false, "Name must be at least " .. minName .. " characters"
    end

    if #name > maxName then
        return false, "Name cannot exceed " .. maxName .. " characters"
    end

    -- Validate description length
    local desc = payload.description or ""
    local minDesc = ix.config.Get("minDescriptionLength", 16)

    if #desc < minDesc then
        return false, "Description must be at least " .. minDesc .. " characters"
    end

    return true
end

-- Setup new character with config defaults
function PLUGIN:OnCharacterCreated(client, character)
    -- Set starting money
    local startMoney = ix.config.Get("defaultMoney", 0)
    character:SetMoney(startMoney)

    -- Create inventory with configured size
    local width = ix.config.Get("inventoryWidth", 6)
    local height = ix.config.Get("inventoryHeight", 4)

    ix.inventory.Create(width, height, character:GetID(), function(inventory)
        character:SetInventory(inventory)
    end)
end

-- Custom attribute system respecting max
function PLUGIN:IncreaseAttribute(character, attribName, amount)
    local current = character:GetAttribute(attribName, 0)
    local max = ix.config.Get("maxAttributes", 100)
    local new = math.min(current + amount, max)

    character:SetAttribute(attribName, new)
end
```

## Best Practices

### ✅ DO

- Use `ix.config.Get()` for all character-related limits
- Always provide default values to `Get()` calls
- Validate character data against config constraints
- Let admins adjust limits through F1 menu
- Check config values before character operations
- Use callbacks to update players when speeds change

### ❌ DON'T

- Don't hardcode character limits or constraints
- Don't bypass config system with custom tables
- Don't forget to validate against min/max values
- Don't modify speeds without checking config
- Don't create characters that exceed configured limits
- Don't ignore saveInterval for manual saves

## Common Patterns

### Pattern 1: Character Validation

```lua
-- Always validate against config before character creation
function ValidateCharacterData(payload)
    local minName = ix.config.Get("minNameLength", 4)
    local maxName = ix.config.Get("maxNameLength", 32)
    local minDesc = ix.config.Get("minDescriptionLength", 16)

    -- Validate all fields
    if #payload.name < minName or #payload.name > maxName then
        return false, "Invalid name length"
    end

    if #payload.description < minDesc then
        return false, "Description too short"
    end

    return true
end
```

### Pattern 2: Dynamic Speed Modification

```lua
-- Modify speed based on config and character state
function PLUGIN:PlayerSpawn(client)
    local character = client:GetCharacter()
    if not character then return end

    -- Get base speeds from config
    local walkSpeed = ix.config.Get("walkSpeed", 130)
    local runSpeed = ix.config.Get("runSpeed", 235)

    -- Apply faction-specific modifiers
    local faction = ix.faction.Get(character:GetFaction())
    if faction.speedMultiplier then
        walkSpeed = walkSpeed * faction.speedMultiplier
        runSpeed = runSpeed * faction.speedMultiplier
    end

    client:SetWalkSpeed(walkSpeed)
    client:SetRunSpeed(runSpeed)
end
```

## Common Issues

### Issue: Characters Exceed Maximum Count

**Cause**: Not checking maxCharacters before allowing creation
**Fix**: Always validate against config before character creation

```lua
-- Check before allowing creation
function SCHEMA:CanPlayerCreateCharacter(client)
    local current = #client:GetCharacters()
    local max = ix.config.Get("maxCharacters", 5)

    if current >= max then
        return false, "Character limit reached"
    end
end
```

### Issue: Speed Changes Not Applied

**Cause**: Config callbacks only run when changed through admin menu
**Fix**: Set speeds on spawn using config values

```lua
-- Apply speeds when player spawns
function PLUGIN:PlayerSpawn(client)
    client:SetWalkSpeed(ix.config.Get("walkSpeed", 130))
    client:SetRunSpeed(ix.config.Get("runSpeed", 235))
end
```

## See Also

- [Configuration System](../systems/configuration.md) - Overview of config system
- [Character System](../systems/character.md) - Character management
- [Inventory System](../systems/inventory.md) - Inventory management
- [Attributes System](../systems/attributes.md) - Character attributes
- [Server Configuration](server.md) - Server-wide settings
- [Inventory Configuration](inventory.md) - Inventory-specific settings
- Source: `gamemode/config/sh_config.lua`
