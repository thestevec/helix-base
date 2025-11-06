# Currency API (ix.currency)

> **Reference**: `gamemode/core/libs/sh_currency.lua`

The currency API manages the server's money system including formatting, spawning money entities, and character money operations. It provides a consistent way to handle in-game economy across all schemas.

## ⚠️ Important: Use Built-in Helix Currency System

**Always use Helix's built-in currency functions** rather than creating custom money systems. The framework automatically provides:
- Customizable currency symbols, names, and models
- Formatted string display (e.g., "$500 dollars")
- Physical money entity spawning
- Character money management with logging
- Automatic money pickup handling
- Network synchronization

## Core Concepts

### What is Currency?

The currency system defines how money appears and behaves:
- Set globally with `ix.currency.Set()`
- Default is dollars with "$" symbol
- Can be customized per schema (caps, credits, etc.)
- Money can exist as character balance or physical entities
- All money operations are logged automatically

### Key Terms

**Symbol**: Prefix shown before amount (e.g., "$", "€", "₽")
**Singular**: Name when amount is 1 (e.g., "dollar", "cap")
**Plural**: Name when amount is more than 1 (e.g., "dollars", "caps")
**Model**: Physical model for spawned money entities

## Library Tables

### ix.currency Properties

**Reference**: `gamemode/core/libs/sh_currency.lua:6-9`

**Realm**: Shared

```lua
ix.currency.symbol    -- Currency symbol (default: "$")
ix.currency.singular  -- Singular name (default: "dollar")
ix.currency.plural    -- Plural name (default: "dollars")
ix.currency.model     -- Entity model (default: "models/props_lab/box01a.mdl")
```

**Example**:
```lua
-- Check current currency
print(ix.currency.symbol)    -- "$"
print(ix.currency.singular)  -- "dollar"
print(ix.currency.plural)    -- "dollars"
```

## Library Functions

### ix.currency.Set

**Reference**: `gamemode/core/libs/sh_currency.lua:17`

**Realm**: Shared

```lua
ix.currency.Set(symbol, singular, plural, model)
```

Sets the currency type for your server.

**Parameters**:
- `symbol` (string) - Currency symbol prefix
- `singular` (string) - Singular form name
- `plural` (string) - Plural form name
- `model` (string) - Model path for money entity

**Example - Fallout Caps**:
```lua
-- In schema/sh_schema.lua
ix.currency.Set("", "cap", "caps", "models/props_junk/PopCan01a.mdl")

-- Results in: "5 caps", "1 cap"
```

**Example - Euros**:
```lua
ix.currency.Set("€", "euro", "euros", "models/props_lab/box01a.mdl")

-- Results in: "€50 euros", "€1 euro"
```

**Example - Credits**:
```lua
ix.currency.Set("", "credit", "credits", "models/props_combine/breenclock.mdl")

-- Results in: "100 credits", "1 credit"
```

**Example - Rubles**:
```lua
ix.currency.Set("₽", "ruble", "rubles", "models/props_lab/box01a.mdl")

-- Results in: "₽1000 rubles"
```

**Note**: Call this in your schema's initialization (usually `sh_schema.lua`).

### ix.currency.Get

**Reference**: `gamemode/core/libs/sh_currency.lua:28`

**Realm**: Shared

```lua
local formatted = ix.currency.Get(amount)
```

Returns a formatted currency string.

**Parameters**:
- `amount` (number) - Money amount

**Returns**: (string) Formatted string with symbol and name

**Example**:
```lua
-- Default currency (dollars)
print(ix.currency.Get(1))     -- "$1 dollar"
print(ix.currency.Get(100))   -- "$100 dollars"
print(ix.currency.Get(0))     -- "$0 dollars"

-- Custom currency (caps)
ix.currency.Set("", "cap", "caps", "models/props_junk/PopCan01a.mdl")
print(ix.currency.Get(5))     -- "5 caps"
print(ix.currency.Get(1))     -- "1 cap"

-- Usage in commands
client:Notify("You received " .. ix.currency.Get(500))
client:Notify("Cost: " .. ix.currency.Get(character:GetMoney()))
```

### ix.currency.Spawn

**Reference**: `gamemode/core/libs/sh_currency.lua:42`

**Realm**: Shared (usually server)

```lua
local entity = ix.currency.Spawn(pos, amount, angle)
```

Spawns a physical money entity in the world.

**Parameters**:
- `pos` (Vector or Player) - Position to spawn at (or player to drop from)
- `amount` (number) - Amount of money
- `angle` (Angle, optional) - Angle of entity (default: angle_zero)

**Returns**: (Entity) Spawned ix_money entity

**Example**:
```lua
-- Spawn at position
local moneyEnt = ix.currency.Spawn(Vector(0, 0, 0), 500)

-- Spawn at player position
ix.currency.Spawn(client, 100)

-- Spawn with custom angle
ix.currency.Spawn(pos, 250, Angle(0, 90, 0))

-- Drop money when player dies
hook.Add("PlayerDeath", "DropMoney", function(victim)
    local character = victim:GetCharacter()
    if character then
        local money = character:GetMoney()
        if money > 0 then
            ix.currency.Spawn(victim:GetPos() + Vector(0, 0, 32), money)
            character:SetMoney(0)
        end
    end
end)
```

**Note**: Returns nil if invalid position or negative amount.

## Character Methods

### character:HasMoney

**Reference**: `gamemode/core/libs/sh_currency.lua:80`

**Realm**: Shared

```lua
local has = character:HasMoney(amount)
```

Checks if character has at least the specified amount.

**Parameters**:
- `amount` (number) - Amount to check

**Returns**: (bool) Whether character has enough money

**Example**:
```lua
-- Check before purchase
if not character:HasMoney(500) then
    client:Notify("You need " .. ix.currency.Get(500))
    return
end

-- Transaction validation
if character:HasMoney(price) then
    character:TakeMoney(price)
    -- Give item
else
    client:Notify("Insufficient funds")
end
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't compare directly
if character:GetMoney() >= amount then  -- Use HasMoney instead
```

### character:GiveMoney

**Reference**: `gamemode/core/libs/sh_currency.lua:88`

**Realm**: Server

```lua
character:GiveMoney(amount, bNoLog)
```

Adds money to a character's balance.

**Parameters**:
- `amount` (number) - Amount to give (negative values converted to positive)
- `bNoLog` (bool, optional) - Skip logging this transaction

**Example**:
```lua
-- Give money
character:GiveMoney(500)
client:Notify("You received " .. ix.currency.Get(500))

-- Quest reward
character:GiveMoney(1000)
client:Notify("Quest completed! +" .. ix.currency.Get(1000))

-- Silent transaction (no log)
character:GiveMoney(100, true)
```

**Note**: Automatically logs to ix.log unless `bNoLog = true`.

### character:TakeMoney

**Reference**: `gamemode/core/libs/sh_currency.lua:100`

**Realm**: Server

```lua
character:TakeMoney(amount, bNoLog)
```

Removes money from a character's balance.

**Parameters**:
- `amount` (number) - Amount to take (negative values converted to positive)
- `bNoLog` (bool, optional) - Skip logging this transaction

**Example**:
```lua
-- Take money
if character:HasMoney(500) then
    character:TakeMoney(500)
    client:Notify("Paid " .. ix.currency.Get(500))
end

-- Purchase item
local price = 250
if character:HasMoney(price) then
    character:TakeMoney(price)
    character:GetInventory():Add("health_kit")
else
    client:Notify("Cannot afford " .. ix.currency.Get(price))
end

-- Silent deduction (no log)
character:TakeMoney(50, true)
```

**⚠️ Warning**: Does not check if character has enough money. Use `HasMoney()` first!

## Complete Examples

### Shop System

```lua
local ITEMS = {
    health_kit = {price = 250, name = "Health Kit"},
    armor_vest = {price = 500, name = "Armor Vest"},
    medkit = {price = 1000, name = "Medkit"}
}

ix.command.Add("Buy", {
    description = "Buy an item from the shop",
    arguments = ix.type.string,
    OnRun = function(self, client, itemID)
        local character = client:GetCharacter()
        local item = ITEMS[itemID]

        if not item then
            return "Item not found"
        end

        if not character:HasMoney(item.price) then
            return "You need " .. ix.currency.Get(item.price)
        end

        -- Process purchase
        character:TakeMoney(item.price)
        character:GetInventory():Add(itemID)

        client:Notify("Purchased " .. item.name .. " for " .. ix.currency.Get(item.price))
    end
})
```

### Money Transfer Command

```lua
ix.command.Add("GiveMoney", {
    description = "Give money to another player",
    arguments = {
        ix.type.player,
        ix.type.number
    },
    OnRun = function(self, client, target, amount)
        local character = client:GetCharacter()
        local targetChar = target:GetCharacter()

        if not targetChar then
            return "Target has no character"
        end

        if amount <= 0 then
            return "Amount must be positive"
        end

        if not character:HasMoney(amount) then
            return "You don't have " .. ix.currency.Get(amount)
        end

        -- Transfer money
        character:TakeMoney(amount)
        targetChar:GiveMoney(amount)

        client:Notify("Sent " .. ix.currency.Get(amount) .. " to " .. target:Name())
        target:Notify("Received " .. ix.currency.Get(amount) .. " from " .. client:Name())
    end
})
```

### Salary System

```lua
-- Pay salaries every 10 minutes
timer.Create("PaySalaries", 600, 0, function()
    for _, client in ipairs(player.GetAll()) do
        local character = client:GetCharacter()
        if not character then continue end

        -- Get salary based on faction/class
        local faction = ix.faction.indices[character:GetFaction()]
        local salary = faction.salary or 100

        character:GiveMoney(salary)
        client:Notify("Received salary: " .. ix.currency.Get(salary))
    end
end)
```

### ATM System

```lua
-- Spawn money at ATM location
function SpawnATMWithdrawal(client, amount)
    local character = client:GetCharacter()

    if not character:HasMoney(amount) then
        client:Notify("Insufficient funds")
        return
    end

    -- Deduct from character
    character:TakeMoney(amount)

    -- Spawn physical money
    local pos = client:GetEyeTrace().HitPos
    ix.currency.Spawn(pos, amount)

    client:Notify("Withdrew " .. ix.currency.Get(amount))
end

-- Pick up money to deposit
hook.Add("OnPickupMoney", "ATMDeposit", function(client, moneyEntity)
    -- Default behavior: add to character balance
    -- This hook is already implemented by framework
end)
```

### Custom Currency for Schema

```lua
-- In schema/sh_schema.lua
function Schema:Initialize()
    -- Set Fallout-style caps currency
    ix.currency.Set("", "cap", "caps", "models/props_junk/PopCan01a.mdl")
end

-- Now all money displays as caps:
-- character:GiveMoney(100)  -- "You received 100 caps"
-- ix.currency.Get(1)        -- "1 cap"
```

## Best Practices

### ✅ DO

- Always use `ix.currency.Get()` for displaying money
- Check `HasMoney()` before `TakeMoney()`
- Use `GiveMoney()` and `TakeMoney()` for balance changes
- Set currency in schema initialization
- Use descriptive currency names that fit your theme
- Log important transactions (default behavior)
- Validate amounts before transactions

### ❌ DON'T

- Don't use `character:SetMoney()` for transactions - use Give/Take
- Don't forget to check HasMoney before TakeMoney
- Don't allow negative transaction amounts
- Don't bypass the currency system with custom variables
- Don't change currency mid-game (set once at startup)
- Don't spawn money with invalid positions
- Don't modify character money without logging (unless intentional)

## Common Patterns

### Transaction Validation

```lua
function ProcessTransaction(character, amount, reason)
    if amount <= 0 then
        return false, "Invalid amount"
    end

    if not character:HasMoney(amount) then
        return false, "Insufficient funds"
    end

    character:TakeMoney(amount)
    return true, "Success"
end
```

### Money Drop on Death

```lua
hook.Add("PlayerDeath", "DropMoneyOnDeath", function(victim, inflictor, attacker)
    local character = victim:GetCharacter()
    if not character then return end

    local money = character:GetMoney()
    if money > 0 then
        -- Drop 50% of money
        local dropAmount = math.floor(money * 0.5)
        ix.currency.Spawn(victim:GetPos() + Vector(0, 0, 32), dropAmount)
        character:TakeMoney(dropAmount)
    end
end)
```

### Periodic Income

```lua
timer.Create("WelfareCheck", 900, 0, function()
    for _, client in ipairs(player.GetAll()) do
        local char = client:GetCharacter()
        if char then
            char:GiveMoney(50)
            client:Notify("Received welfare: " .. ix.currency.Get(50))
        end
    end
end)
```

## Common Issues

### Money Not Saving

**Cause**: Using SetMoney without saving, or server crash.

**Fix**: Framework auto-saves. Use Give/TakeMoney:
```lua
-- These auto-save
character:GiveMoney(100)
character:TakeMoney(50)
```

### Negative Money Amounts

**Cause**: TakeMoney more than character has.

**Fix**: Always check first:
```lua
-- WRONG
character:TakeMoney(999999)  -- Character can go negative!

-- CORRECT
if character:HasMoney(price) then
    character:TakeMoney(price)
else
    client:Notify("Insufficient funds")
end
```

### Currency Format Not Changing

**Cause**: Setting currency after clients load or client cache.

**Fix**: Set currency early in schema:
```lua
-- In schema/sh_schema.lua
function Schema:Initialize()
    ix.currency.Set("", "cap", "caps", "models/props_junk/PopCan01a.mdl")
end
```

## Related Hooks

### OnPickupMoney

Called when player picks up money entity.

```lua
-- Already implemented by framework
function GM:OnPickupMoney(client, moneyEntity)
    if IsValid(moneyEntity) then
        local amount = moneyEntity:GetAmount()
        client:GetCharacter():GiveMoney(amount)
    end
end

-- Override for custom behavior
hook.Add("OnPickupMoney", "CustomPickup", function(client, moneyEntity)
    -- Custom logic
    return true  -- Prevent default behavior
end)
```

### PlayerDeath

Use to handle money drops on death.

```lua
hook.Add("PlayerDeath", "MoneyDrop", function(victim)
    local char = victim:GetCharacter()
    if char and char:GetMoney() > 0 then
        ix.currency.Spawn(victim, char:GetMoney())
        char:SetMoney(0)
    end
end)
```

## See Also

- [Character API](character.md) - Character money methods
- [Config API](config.md) - Configuring money limits
- [Command API](command.md) - Creating money commands
- [Log API](log.md) - Transaction logging
- Source: `gamemode/core/libs/sh_currency.lua`
