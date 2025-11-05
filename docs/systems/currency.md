# Currency System (ix.currency)

> **Reference**: `gamemode/core/libs/sh_currency.lua`

The currency system manages in-game money, including display formatting, entity spawning, and character money transactions. It provides a unified way to handle currency across different schemas without hardcoding currency types.

## ⚠️ Important: Use Built-in Helix Currency

**Always use Helix's currency system** rather than creating custom money implementations. The framework provides:
- Automatic currency formatting with symbols and names
- Money entity spawning and pickup handling
- Character money methods (GiveMoney, TakeMoney, HasMoney)
- Transaction logging integration
- Database persistence through character system
- Customizable currency types per schema

## Core Concepts

### What is the Currency System?

The currency system handles three main aspects:
1. **Display Formatting**: Converting numeric amounts into readable strings (e.g., "$100 dollars")
2. **Entity Spawning**: Creating physical money entities in the world (`ix_money`)
3. **Character Transactions**: Giving, taking, and checking money on characters

### Currency Customization

Schemas can customize their currency by setting four properties:
- **Symbol**: The currency symbol (e.g., `"$"`, `"£"`, `"₽"`)
- **Singular**: Singular form of currency name (e.g., `"dollar"`, `"token"`)
- **Plural**: Plural form of currency name (e.g., `"dollars"`, `"tokens"`)
- **Model**: World model for dropped money entities

### Default Values

```lua
ix.currency.symbol = "$"
ix.currency.singular = "dollar"
ix.currency.plural = "dollars"
ix.currency.model = "models/props_lab/box01a.mdl"
```

## Using the Currency System

### ix.currency.Set

**Reference**: `gamemode/core/libs/sh_currency.lua:17`

Configures the server's currency type. Should be called in schema initialization.

```lua
-- Set currency to US Dollars (default)
ix.currency.Set("$", "dollar", "dollars", "models/props_lab/box01a.mdl")

-- Set currency to Tokens
ix.currency.Set("", "token", "tokens", "models/props_junk/popcan01a.mdl")

-- Set currency to Euros
ix.currency.Set("€", "euro", "euros", "models/props_lab/box01a.mdl")

-- Set currency to Rubles
ix.currency.Set("₽", "ruble", "rubles", "models/props_lab/box01a.mdl")

-- Set currency to Caps (Fallout-style)
ix.currency.Set("", "cap", "caps", "models/props_junk/PopCan01a.mdl")
```

**Parameters**:
- `symbol` (string): Currency symbol (can be empty string)
- `singular` (string): Singular name
- `plural` (string): Plural name
- `model` (string): World model path

**Usage Location**: Call in `schema/sh_schema.lua` or `sh_config.lua`

### ix.currency.Get

**Reference**: `gamemode/core/libs/sh_currency.lua:28`

Formats a numeric amount into a human-readable currency string.

```lua
-- Format currency amounts
print(ix.currency.Get(1))    -- "$1 dollar"
print(ix.currency.Get(100))  -- "$100 dollars"
print(ix.currency.Get(0))    -- "$0 dollars"

-- Use in notifications
client:Notify("You received " .. ix.currency.Get(50))
-- "You received $50 dollars"

-- Use in UI
local text = "Balance: " .. ix.currency.Get(character:GetMoney())
-- "Balance: $250 dollars"
```

**Parameters**:
- `amount` (number): Numeric amount to format

**Returns**:
- (string): Formatted currency string

**Note**: Automatically uses singular form for amount of 1, plural otherwise.

### ix.currency.Spawn

**Reference**: `gamemode/core/libs/sh_currency.lua:42`

Spawns a physical money entity in the world that players can pick up.

```lua
-- Spawn money at position
local pos = Vector(0, 0, 0)
local money = ix.currency.Spawn(pos, 100)

-- Spawn with angle
local angle = Angle(0, 90, 0)
local money = ix.currency.Spawn(pos, 50, angle)

-- Spawn at player position (drops at player's feet)
local money = ix.currency.Spawn(client, 25)

-- Error handling
local money = ix.currency.Spawn(pos, 100)
if (IsValid(money)) then
    print("Money spawned successfully")
else
    print("Failed to spawn money")
end
```

**Parameters**:
- `pos` (Vector or Player): Position or player to spawn at
- `amount` (number): Amount of money (must be positive)
- `angle` (Angle, optional): Entity angle (defaults to `angle_zero`)

**Returns**:
- (Entity): The spawned `ix_money` entity, or nil if failed

**Behavior**:
- Validates amount (must be positive)
- If `pos` is a player, spawns at player's drop position
- Automatically rounds amount to integer
- Returns nil and prints error if invalid parameters

### Character Money Methods

**Reference**: `gamemode/core/libs/sh_currency.lua:78-111`

The currency system extends the Character class with money management methods.

#### character:GetMoney

**Reference**: `gamemode/core/libs/sh_character.lua:683-693`

Gets the character's current money amount.

```lua
local character = client:GetCharacter()
local money = character:GetMoney()

print("Player has " .. ix.currency.Get(money))
```

**Returns**:
- (number): Current money amount

**Realm**: Shared (server + owning client only)

#### character:HasMoney

**Reference**: `gamemode/core/libs/sh_currency.lua:80`

Checks if character has at least the specified amount.

```lua
local character = client:GetCharacter()

if (character:HasMoney(100)) then
    print("Character can afford this")
else
    client:Notify("You need " .. ix.currency.Get(100))
end
```

**Parameters**:
- `amount` (number): Amount to check

**Returns**:
- (bool): True if character has enough money

#### character:GiveMoney

**Reference**: `gamemode/core/libs/sh_currency.lua:88`

Adds money to the character's balance.

```lua
local character = client:GetCharacter()

-- Give money (with logging)
character:GiveMoney(100)

-- Give money without logging
character:GiveMoney(100, true)  -- bNoLog = true

-- With notification
local amount = 50
character:GiveMoney(amount)
client:Notify("You received " .. ix.currency.Get(amount))
```

**Parameters**:
- `amount` (number): Amount to give (automatically made positive)
- `bNoLog` (bool, optional): If true, skip transaction logging

**Returns**:
- (bool): Always returns true

**Behavior**:
- Automatically converts negative amounts to positive
- Logs transaction unless `bNoLog = true`
- Updates character's money immediately
- Auto-saves to database on next character save

#### character:TakeMoney

**Reference**: `gamemode/core/libs/sh_currency.lua:100`

Removes money from the character's balance.

```lua
local character = client:GetCharacter()

-- Check before taking
if (character:HasMoney(100)) then
    character:TakeMoney(100)
    client:Notify("Purchase successful")
else
    client:Notify("Not enough money")
    return false
end

-- Take without logging
character:TakeMoney(50, true)  -- bNoLog = true
```

**Parameters**:
- `amount` (number): Amount to take (automatically made positive)
- `bNoLog` (bool, optional): If true, skip transaction logging

**Returns**:
- (bool): Always returns true

**⚠️ Warning**: This method does NOT check if character has enough money! Always use `character:HasMoney()` first.

#### character:SetMoney

**Reference**: `gamemode/core/libs/sh_character.lua:681`

Directly sets the character's money to a specific amount (admin command use).

```lua
-- Set exact amount (usually for admin commands only)
character:SetMoney(500)
```

**Parameters**:
- `amount` (number): New total amount

**Note**: Prefer `GiveMoney()` and `TakeMoney()` for gameplay transactions as they provide logging.

## Complete Examples

### Example 1: Shop Purchase System

```lua
-- plugins/myshop/sv_hooks.lua
function PLUGIN:PlayerPurchaseItem(client, itemUniqueName, price)
    local character = client:GetCharacter()

    -- Validate character exists
    if (!character) then
        return false
    end

    -- Check if player has enough money
    if (!character:HasMoney(price)) then
        client:Notify("You need " .. ix.currency.Get(price))
        return false
    end

    -- Take money
    character:TakeMoney(price)

    -- Give item
    character:GetInventory():Add(itemUniqueName)

    -- Notify
    client:Notify("Purchased for " .. ix.currency.Get(price))

    return true
end
```

### Example 2: Salary System

```lua
-- schema/sv_hooks.lua
function Schema:PlayerPaySalary(client)
    local character = client:GetCharacter()
    if (!character) then return end

    local faction = ix.faction.Get(character:GetFaction())
    local salary = faction.salary or 50

    -- Give salary
    character:GiveMoney(salary)

    -- Notify player
    client:NotifyLocalized("salary", ix.currency.Get(salary))
end
```

### Example 3: Money Drop on Death

```lua
-- schema/sv_hooks.lua
function Schema:PlayerDeath(client, inflictor, attacker)
    local character = client:GetCharacter()
    if (!character) then return end

    local money = character:GetMoney()

    -- Drop portion of money (50%)
    if (money > 0) then
        local dropAmount = math.floor(money * 0.5)

        character:TakeMoney(dropAmount)
        ix.currency.Spawn(client:GetPos(), dropAmount)

        client:Notify("You dropped " .. ix.currency.Get(dropAmount))
    end
end
```

### Example 4: Custom Currency Schema

```lua
-- schema/sh_schema.lua
SCHEMA.name = "Fallout RP"
SCHEMA.author = "Author"
SCHEMA.description = "Post-apocalyptic roleplay"

-- Set custom currency (bottle caps)
ix.currency.Set("", "cap", "caps", "models/props_junk/PopCan01a.mdl")

-- Now all currency displays as "caps"
-- character:GiveMoney(100)
-- client:Notify(ix.currency.Get(100)) -- "100 caps"
```

### Example 5: Vendor Transaction

```lua
-- plugins/myvendor/sv_plugin.lua
function PLUGIN:VendorSell(client, vendor, itemName, price)
    local character = client:GetCharacter()

    -- Give money for selling
    character:GiveMoney(price)

    -- Log transaction
    ix.log.Add(client, "vendorSell", itemName, vendor:GetName(),
        ix.currency.Get(price))

    -- Notify
    client:NotifyLocalized("businessSell", itemName, ix.currency.Get(price))
end
```

## Best Practices

### ✅ DO

- Use `ix.currency.Set()` once in schema initialization
- Always use `ix.currency.Get()` to format money for display
- Check `character:HasMoney()` before taking money
- Use `character:GiveMoney()` and `character:TakeMoney()` for transactions
- Let the logging system track transactions (don't pass `bNoLog = true` unless necessary)
- Validate character exists before accessing money methods
- Use `ix.currency.Spawn()` to create physical money entities

### ❌ DON'T

- Don't access `character.vars.money` directly - use `GetMoney()`/`SetMoney()`
- Don't forget to check `HasMoney()` before `TakeMoney()`
- Don't hardcode currency strings like `"$"` or `"dollars"` - use `ix.currency.Get()`
- Don't call `ix.currency.Set()` multiple times or after initialization
- Don't use `SetMoney()` for gameplay transactions - use `GiveMoney()`/`TakeMoney()`
- Don't spawn money with negative amounts
- Don't bypass logging unless you have a specific reason

**⚠️ Do NOT**:
```lua
-- WRONG: Hardcoded currency display
client:Notify("You received $100 dollars")

-- WRONG: Direct variable access
character.vars.money = character.vars.money + 100

-- WRONG: No validation before taking
character:TakeMoney(100)  -- What if they don't have enough?

-- WRONG: Using SetMoney for transactions
character:SetMoney(character:GetMoney() + 100)  -- No logging!

-- CORRECT: Use framework methods
if (character:HasMoney(100)) then
    character:TakeMoney(100)
    client:Notify("Purchased for " .. ix.currency.Get(100))
end
```

## Common Patterns

### Pattern 1: Transaction with Validation

```lua
function PLUGIN:ProcessPayment(client, amount)
    local character = client:GetCharacter()

    -- Validate character
    if (!character) then
        return false, "No character"
    end

    -- Validate amount
    if (amount <= 0) then
        return false, "Invalid amount"
    end

    -- Check funds
    if (!character:HasMoney(amount)) then
        client:Notify("Need " .. ix.currency.Get(amount))
        return false, "Insufficient funds"
    end

    -- Process transaction
    character:TakeMoney(amount)

    return true
end
```

### Pattern 2: Money Pickup Handling

The framework automatically handles money pickup through the `GM:OnPickupMoney` hook:

**Reference**: `gamemode/core/libs/sh_currency.lua:69`

```lua
-- This is handled automatically by the framework
-- Override only if you need custom behavior
function PLUGIN:OnPickupMoney(client, moneyEntity)
    if (IsValid(moneyEntity)) then
        local amount = moneyEntity:GetAmount()
        local character = client:GetCharacter()

        -- Custom logic before pickup
        if (character:GetFaction() == FACTION_THIEF) then
            amount = amount * 1.5  -- Thieves get 50% more
        end

        character:GiveMoney(amount)

        -- Custom notification
        client:Notify("Picked up " .. ix.currency.Get(amount))

        -- Return false to prevent default behavior
        return false
    end
end
```

### Pattern 3: Displaying Money in UI

```lua
-- Client-side UI code
local money = LocalPlayer():GetCharacter():GetMoney()
local formattedMoney = ix.currency.Get(money)

local label = vgui.Create("DLabel", parent)
label:SetText("Balance: " .. formattedMoney)
label:SetFont("ixMediumFont")
```

## Common Issues

### Money Not Saving

**Cause**: Character not being saved after money changes
**Fix**: The framework auto-saves characters periodically. Money is saved with character data automatically.

```lua
-- Manual save if needed (rarely necessary)
character:GiveMoney(100)
character:Save(function()
    print("Character saved with new money amount")
end)
```

### TakeMoney Goes Negative

**Cause**: Not checking `HasMoney()` before taking money
**Fix**: Always validate before taking

```lua
-- ❌ WRONG
character:TakeMoney(1000)  -- Can go negative!

-- ✅ CORRECT
if (character:HasMoney(1000)) then
    character:TakeMoney(1000)
else
    client:Notify("Not enough money")
end
```

### Currency Display Wrong

**Cause**: Using hardcoded strings instead of `ix.currency.Get()`
**Fix**: Always use the formatting function

```lua
-- ❌ WRONG
client:Notify("You received $" .. amount .. " dollars")

-- ✅ CORRECT
client:Notify("You received " .. ix.currency.Get(amount))
```

### Money Entity Not Spawning

**Cause**: Invalid position or negative amount
**Fix**: Validate parameters before spawning

```lua
-- Check for valid position
local pos = client:GetPos()
if (!isvector(pos)) then
    return
end

-- Check for positive amount
local amount = math.max(1, math.floor(amount))

-- Spawn
local money = ix.currency.Spawn(pos, amount)
if (!IsValid(money)) then
    ErrorNoHalt("Failed to spawn money entity\n")
end
```

### GetMoney Returns nil on Client

**Cause**: Trying to access other player's money (only own character synced to client)
**Fix**: Only access money on server, or for local player on client

```lua
-- ❌ WRONG (client trying to get other player's money)
if (CLIENT) then
    local otherMoney = otherPlayer:GetCharacter():GetMoney()  -- nil!
end

-- ✅ CORRECT (server can access any character)
if (SERVER) then
    local otherMoney = otherPlayer:GetCharacter():GetMoney()
end

-- ✅ CORRECT (client accessing own money)
if (CLIENT) then
    local myMoney = LocalPlayer():GetCharacter():GetMoney()
end
```

## See Also

- [Character System](character.md) - Character management and variables
- [Inventory System](inventory.md) - Item transactions and storage
- [Commands](commands.md) - Creating money-related commands
- Source: `gamemode/core/libs/sh_currency.lua`
- Character Source: `gamemode/core/libs/sh_character.lua:687`
- Entity: `entities/entities/ix_money.lua`
