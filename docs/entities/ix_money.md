# Money Entity (ix_money)

> **Reference**: `entities/entities/ix_money.lua`

The `ix_money` entity represents physical currency in the game world. It spawns when players drop money or when money is given via commands.

## ⚠️ Important: Use Built-in Currency Functions

**Always use `ix.currency.Spawn()`** rather than creating ix_money entities directly. The framework provides:
- Automatic amount validation
- Position handling (player or vector)
- Network synchronization
- Pickup handling
- Money logging

## Core Concepts

### What is a Money Entity?

A money entity (`ix_money`) is a physical representation of currency that exists in the game world. Players can pick up money entities to add currency to their character's wallet.

Key features:
- Uses customizable currency model
- Displays amount in tooltip
- Automatic pickup handling
- Ownership tracking (optional)
- Pickup delay for dropped items

### Key Properties

**Entity Variables** (line 10-12):
- `Amount` (Int): The amount of money this entity represents

**Internal Properties** (lines 36-39):
- `ixSteamID`: Steam ID of original owner (if dropped by player)
- `ixCharID`: Character ID of original owner (prevents other characters from same player picking it up)

## Spawning Money Entities

### Using ix.currency.Spawn()

**Reference**: `gamemode/core/libs/sh_currency.lua:42`

```lua
-- Spawn money at a position
local money = ix.currency.Spawn(position, amount, angle)
```

**Complete Working Examples**:

```lua
-- Spawn $500 at a specific position
local spawnPos = Vector(100, 200, 50)
local money = ix.currency.Spawn(spawnPos, 500)

if IsValid(money) then
    print("Spawned $500 at position")
end

-- Spawn money at player's feet
local money = ix.currency.Spawn(client, 250)
-- Framework automatically calculates drop position

-- Spawn money with custom angle
local money = ix.currency.Spawn(spawnPos, 1000, Angle(0, 45, 0))
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't create ix_money entities manually
local money = ents.Create("ix_money")
money:SetPos(position)
money:Spawn()  -- Missing amount, activation, validation!

-- WRONG: Don't spawn negative amounts
ix.currency.Spawn(position, -100)  -- Will be rejected
```

## Currency Configuration

### Setting Currency Type

**Reference**: `gamemode/core/libs/sh_currency.lua:17-22`

```lua
-- In your schema or plugin
function SCHEMA:InitializedConfig()
    -- Set custom currency
    ix.currency.Set(
        "€",           -- Symbol
        "euro",        -- Singular name
        "euros",       -- Plural name
        "models/props/cs_office/money_bag.mdl"  -- Model
    )
end
```

**Default values**:
- Symbol: `$`
- Singular: `dollar`
- Plural: `dollars`
- Model: `models/props_lab/box01a.mdl`

### Formatting Currency

**Reference**: `gamemode/core/libs/sh_currency.lua:28-34`

```lua
-- Get formatted currency string
local formatted = ix.currency.Get(500)
-- Returns: "$500 dollars"

local single = ix.currency.Get(1)
-- Returns: "$1 dollar"

-- Use in notifications
client:Notify("You received " .. ix.currency.Get(amount))
```

## Entity Behavior

### Player Pickup

**Reference**: `entities/entities/ix_money.lua:35-50`

When a player uses (presses E on) a money entity:

```lua
function ENT:Use(activator)
    -- Framework automatically:
    -- 1. Checks ownership (if ixSteamID and ixCharID are set)
    -- 2. Shows interaction progress bar
    -- 3. Adds money to character
    -- 4. Removes entity from world
end
```

**Ownership check** (lines 36-43):
- If money was dropped by a player, it stores their SteamID and CharID
- Other characters from the same player cannot pick it up
- This prevents exploiting by switching characters

### Pickup Hook

**Reference**: `gamemode/core/libs/sh_currency.lua:69-75`

```lua
-- Called when money is picked up
function GM:OnPickupMoney(client, moneyEntity)
    if IsValid(moneyEntity) then
        local amount = moneyEntity:GetAmount()
        client:GetCharacter():GiveMoney(amount)
    end
end
```

**Customizing pickup behavior**:

```lua
-- In your schema or plugin
function SCHEMA:OnPickupMoney(client, moneyEntity)
    local amount = moneyEntity:GetAmount()

    -- Tax system - take 10%
    local tax = math.floor(amount * 0.1)
    local netAmount = amount - tax

    client:GetCharacter():GiveMoney(netAmount)
    client:Notify("You picked up " .. ix.currency.Get(netAmount) .. " (taxed " .. ix.currency.Get(tax) .. ")")

    return false  -- Prevent default behavior
end
```

## Character Money Functions

### Giving Money

**Reference**: `gamemode/core/libs/sh_currency.lua:88-98`

```lua
-- Give money to a character
character:GiveMoney(amount, noLog)
```

**Examples**:

```lua
-- Give money with logging
local character = client:GetCharacter()
character:GiveMoney(500)
-- Logs the transaction

-- Give money without logging
character:GiveMoney(500, true)
-- Silent transaction, no log entry
```

### Taking Money

**Reference**: `gamemode/core/libs/sh_currency.lua:100-110`

```lua
-- Take money from a character
character:TakeMoney(amount, noLog)
```

**Examples**:

```lua
-- Take money with logging
character:TakeMoney(250)

-- Check first, then take
if character:HasMoney(250) then
    character:TakeMoney(250)
    client:Notify("Purchase successful!")
else
    client:Notify("You don't have enough money!")
end
```

### Checking Money

**Reference**: `gamemode/core/libs/sh_currency.lua:80-86`

```lua
-- Check if character has enough money
if character:HasMoney(amount) then
    -- Character can afford it
end
```

**Examples**:

```lua
-- Simple check
if character:HasMoney(1000) then
    client:Notify("You can afford this!")
end

-- Get current money
local currentMoney = character:GetMoney()
client:Notify("You have " .. ix.currency.Get(currentMoney))
```

## Client-Side Features

### Tooltip Display

**Reference**: `entities/entities/ix_money.lua:58-63`

```lua
function ENT:OnPopulateEntityInfo(container)
    -- Shows formatted currency amount
    -- Example: "$500 dollars"
end
```

The tooltip automatically uses the configured currency format.

## Complete Examples

### Dropping Money

```lua
-- Command to drop money
ix.command.Add("DropMoney", {
    description = "Drop money on the ground.",
    arguments = ix.type.number,
    OnRun = function(self, client, amount)
        local character = client:GetCharacter()

        if not character then
            return "@notNow"
        end

        amount = math.floor(amount)

        if amount <= 0 then
            return "@invalidArg", 1
        end

        if not character:HasMoney(amount) then
            return "@notEnoughMoney"
        end

        character:TakeMoney(amount)

        -- Spawn money entity
        local money = ix.currency.Spawn(client, amount)

        if IsValid(money) then
            -- Store ownership to prevent exploitation
            money.ixSteamID = client:SteamID()
            money.ixCharID = character:GetID()

            client:NotifyLocalized("moneyTaken", ix.currency.Get(amount))
        end
    end
})
```

### Money Reward System

```lua
-- Give money rewards for completing tasks
function PLUGIN:OnTaskCompleted(client, taskID)
    local rewards = {
        ["task_delivery"] = 500,
        ["task_repair"] = 750,
        ["task_combat"] = 1000
    }

    local amount = rewards[taskID]

    if amount then
        local character = client:GetCharacter()

        if character then
            character:GiveMoney(amount)
            client:Notify("You earned " .. ix.currency.Get(amount) .. "!")

            -- Optionally spawn physical money instead
            -- ix.currency.Spawn(client:GetPos() + Vector(0, 0, 50), amount)
        end
    end
end
```

### Money Fountain/Spawner

```lua
-- Spawn money periodically at a location
hook.Add("Initialize", "MoneyFountain", function()
    timer.Create("MoneyFountain", 300, 0, function()
        local spawnPos = Vector(1000, 500, 100)
        local amount = math.random(50, 200)

        local money = ix.currency.Spawn(spawnPos, amount)

        if IsValid(money) then
            -- Add some upward velocity
            local phys = money:GetPhysicsObject()

            if IsValid(phys) then
                phys:SetVelocity(Vector(0, 0, 200))
            end
        end
    end)
end)
```

### ATM System

```lua
-- Simple ATM withdrawal
function OpenATM(client)
    local character = client:GetCharacter()

    if not character then
        return
    end

    ix.util.StringRequest("Withdraw Amount", "How much money do you want to withdraw?", function(text)
        local amount = tonumber(text)

        if not amount or amount <= 0 then
            client:Notify("Invalid amount!")
            return
        end

        -- Check bank balance (you'd implement banking separately)
        local bankBalance = character:GetData("bankMoney", 0)

        if bankBalance < amount then
            client:Notify("Insufficient funds in bank!")
            return
        end

        -- Deduct from bank
        character:SetData("bankMoney", bankBalance - amount)

        -- Spawn money at ATM position
        local atmPos = client:GetPos() + client:GetForward() * 50

        ix.currency.Spawn(atmPos, amount)

        client:Notify("Withdrew " .. ix.currency.Get(amount))
    end, "100")
end
```

## Best Practices

### ✅ DO

- Use `ix.currency.Spawn()` to create money entities
- Use `character:HasMoney()` before taking money
- Use `character:GiveMoney()` and `character:TakeMoney()` for transactions
- Validate amounts before spawning (positive integers)
- Store ownership info when dropping money (`ixSteamID`, `ixCharID`)
- Use `ix.currency.Get()` for formatted display

### ❌ DON'T

- Don't create `ix_money` entities with `ents.Create()`
- Don't forget to check `character:HasMoney()` before purchases
- Don't spawn negative amounts
- Don't bypass the currency system with direct money manipulation
- Don't forget to log important transactions
- Don't allow money duplication exploits

## Common Patterns

### Transaction with Validation

```lua
-- Safe transaction pattern
local function BuyItem(client, price)
    local character = client:GetCharacter()

    if not character then
        return false, "No character loaded"
    end

    if not character:HasMoney(price) then
        return false, "Not enough money"
    end

    character:TakeMoney(price)
    return true
end

-- Usage
local success, error = BuyItem(client, 500)

if not success then
    client:Notify(error)
end
```

### Money Drop on Death

```lua
-- Drop money when player dies
function SCHEMA:PlayerDeath(client)
    local character = client:GetCharacter()

    if character then
        local money = character:GetMoney()

        if money > 0 then
            -- Drop percentage of money
            local dropAmount = math.floor(money * 0.5)

            character:TakeMoney(dropAmount)

            -- Spawn at death position
            local pos = client:GetPos()
            ix.currency.Spawn(pos + Vector(0, 0, 10), dropAmount)
        end
    end
end
```

### Shared Money Pool

```lua
-- Money entity that multiple players can take from
local poolMoney = ix.currency.Spawn(position, 1000)

if IsValid(poolMoney) then
    -- Don't set ownership - anyone can pick it up
    -- poolMoney.ixSteamID = nil
    -- poolMoney.ixCharID = nil
end
```

## Common Issues

### Money Entity Doesn't Spawn

**Cause**: Invalid amount or position
**Fix**: Validate inputs before calling `ix.currency.Spawn()`

```lua
-- Validate first
if amount > 0 and isvector(position) then
    ix.currency.Spawn(position, amount)
end
```

### Can't Pick Up Own Money

**Cause**: Ownership check preventing pickup with different character
**Fix**: This is intentional to prevent exploits. Use same character or clear ownership.

```lua
-- To allow any character to pick up
money.ixSteamID = nil
money.ixCharID = nil
```

### Money Amount Shows Wrong Currency

**Cause**: Currency not configured in schema
**Fix**: Set currency in schema initialization

```lua
function SCHEMA:InitializedConfig()
    ix.currency.Set("£", "pound", "pounds", "models/props/cs_office/money_bag.mdl")
end
```

### Money Entity Falls Through Floor

**Cause**: Spawned before physics initialized or invalid position
**Fix**: Use `ix.currency.Spawn()` which handles activation properly

```lua
-- Framework handles this automatically
local money = ix.currency.Spawn(position, amount)
-- money:Activate() is called internally
```

## See Also

- [Configuration System](../systems/configuration.md) - Setting currency options
- [Character System](../systems/character.md) - Character money management
- [Command System](../systems/commands.md) - Creating money commands
- [ix_item Entity](ix_item.md) - Item entity documentation
- Source: `entities/entities/ix_money.lua`
- Source: `gamemode/core/libs/sh_currency.lua`
