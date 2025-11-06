# Date & Time System (ix.date)

> **Reference**: `gamemode/core/libs/sh_date.lua`

The date and time system provides persistent, customizable in-game date/time tracking that goes beyond Unix epoch limitations (pre-1970), with configurable time scales and automatic synchronization.

## ⚠️ Important: Use Built-in Date System

**Always use Helix's date system** rather than Lua's standard time functions for in-game time. The framework provides:
- Dates before 1970 (not limited by Unix epoch)
- Configurable time scale (e.g., 1 real second = 1 in-game minute)
- Automatic persistence across server restarts
- Network synchronization to all clients
- Third-party date library with extensive formatting options
- Integration with config system

## Core Concepts

### What is the Date System?

Helix's date system overcomes Lua's Unix epoch limitation by using a third-party date library that represents time as objects rather than seconds since 1970. This allows roleplaying servers to set their game world in any time period.

**Key Features**:
- Set game world date to any year (e.g., 1942, 2077, 3020)
- Control time progression speed
- Persist date across server restarts
- Format dates in multiple ways
- Synchronize to all connected clients

### Key Terms

- **Date Object**: An object representing a specific point in time with methods for manipulation
- **Time Scale**: How many real-life seconds equal one in-game minute
- **Serialization**: Converting date object to table for storage/networking
- **Offset**: Time difference since last sync point

### Understanding Time Scale

**Reference**: `gamemode/core/libs/sh_date.lua:17`

Time scale determines how fast in-game time progresses:
- `60` = Real-time (60 seconds = 1 minute)
- `6` = 10x speed (6 seconds = 1 minute)
- `1` = 60x speed (1 second = 1 minute)
- `120` = 0.5x speed (120 seconds = 1 minute)

## Using the Date System

### ix.date.Get

**Reference**: `gamemode/core/libs/sh_date.lua:117`

```lua
local dateObj = ix.date.Get()
```

Returns the current in-game date as a date object with methods.

**Complete Example**:
```lua
-- Get current date
local currentDate = ix.date.Get()

-- Access date components
local year = currentDate:getyear()
local month = currentDate:getmonth()
local day = currentDate:getday()
local hour = currentDate:gethour()
local minute = currentDate:getminutes()
local second = currentDate:getseconds()

print(string.format("Date: %d-%02d-%02d %02d:%02d:%02d",
    year, month, day, hour, minute, second))

-- Date arithmetic
local tomorrow = ix.date.Get():adddays(1)
local nextWeek = ix.date.Get():adddays(7)
local lastMonth = ix.date.Get():addmonths(-1)
local nextYear = ix.date.Get():addyears(1)

-- Compare dates
local futureDate = ix.date.Get():adddays(5)
if futureDate > ix.date.Get() then
    print("Future date is later than now")
end

-- Day of week
local dayOfWeek = currentDate:getweekday()  -- 1 = Sunday, 7 = Saturday
local dayName = currentDate:fmt("%A")  -- "Monday", "Tuesday", etc.
```

**⚠️ Do NOT**:
```lua
-- WRONG: Don't use os.time() for in-game time
local time = os.time()  -- This is real-world time, not game time!

-- WRONG: Don't use os.date() for game dates
local date = os.date("%Y-%m-%d")  -- Real-world date, not in-game!

-- WRONG: Don't calculate time manually
local gameTime = CurTime() * someScale  -- Use ix.date.Get()!
```

### ix.date.GetFormatted

**Reference**: `gamemode/core/libs/sh_date.lua:128`

```lua
local formatted = ix.date.GetFormatted(format, dateObj)
```

Returns a formatted string representation of a date using standard strftime format codes.

**Parameters**:
- `format` (string): Format string (e.g., `"%Y-%m-%d %H:%M:%S"`)
- `dateObj` (date, optional): Date to format (defaults to current date)

**Common Format Codes**:
- `%Y` - 4-digit year (2024)
- `%y` - 2-digit year (24)
- `%m` - Month as number (01-12)
- `%B` - Full month name (January)
- `%b` - Short month name (Jan)
- `%d` - Day of month (01-31)
- `%A` - Full day name (Monday)
- `%a` - Short day name (Mon)
- `%H` - Hour 24-hour (00-23)
- `%I` - Hour 12-hour (01-12)
- `%M` - Minute (00-59)
- `%S` - Second (00-59)
- `%p` - AM/PM

**Complete Example**:
```lua
-- Display current date in various formats
local date1 = ix.date.GetFormatted("%Y-%m-%d")  -- "2024-11-05"
local date2 = ix.date.GetFormatted("%B %d, %Y")  -- "November 05, 2024"
local date3 = ix.date.GetFormatted("%d/%m/%Y %H:%M")  -- "05/11/2024 14:30"
local date4 = ix.date.GetFormatted("%A, %B %d, %Y at %I:%M %p")
    -- "Tuesday, November 05, 2024 at 02:30 PM"

-- Format specific date
local specificDate = ix.date.Get():adddays(7)
local futureStr = ix.date.GetFormatted("%B %d", specificDate)

-- Update UI with formatted time
function PANEL:Paint(w, h)
    local timeStr = ix.date.GetFormatted("%I:%M %p")
    draw.SimpleText(timeStr, "MyFont", w/2, 10, color_white, TEXT_ALIGN_CENTER)
end

-- Show date in chat
function PLUGIN:ShowDate(client)
    local dateStr = ix.date.GetFormatted("%A, %B %d, %Y")
    client:Notify("Today is " .. dateStr)
end

-- Create timestamp for logs
function PLUGIN:LogWithTimestamp(message)
    local timestamp = ix.date.GetFormatted("[%Y-%m-%d %H:%M:%S]")
    print(timestamp .. " " .. message)
end
```

### ix.date.GetSerialized

**Reference**: `gamemode/core/libs/sh_date.lua:136`

```lua
local serialized = ix.date.GetSerialized(dateObj)
```

Converts a date object to a table for networking or storage.

**Complete Example**:
```lua
-- Serialize current date
local dateTable = ix.date.GetSerialized()

-- Network date to clients
net.Start("MyDateMessage")
    net.WriteTable(dateTable)
net.Send(client)

-- Save date to character data
function PLUGIN:SaveLastLoginTime(character)
    local loginDate = ix.date.GetSerialized()
    character:SetData("lastLogin", loginDate)
end

-- Compare saved date
function PLUGIN:CheckInactivity(character)
    local lastLogin = character:GetData("lastLogin")

    if lastLogin then
        local lastDate = ix.date.Construct(lastLogin)
        local currentDate = ix.date.Get()
        local daysDiff = currentDate:spandays(lastDate)

        if daysDiff > 30 then
            print("Character has been inactive for " .. daysDiff .. " days")
        end
    end
end

-- Store date in database
function PLUGIN:SaveEventDate(eventName, eventDate)
    local serialized = ix.date.GetSerialized(eventDate)

    ix.db.Query(
        "INSERT INTO events (name, date_data) VALUES (?, ?)",
        eventName,
        util.TableToJSON(serialized)
    )
end
```

### ix.date.Construct

**Reference**: `gamemode/core/libs/sh_date.lua:144`

```lua
local dateObj = ix.date.Construct(serializedDate)
```

Converts a serialized date table back into a date object.

**Complete Example**:
```lua
-- Receive networked date
net.Receive("MyDateMessage", function()
    local dateTable = net.ReadTable()
    local dateObj = ix.date.Construct(dateTable)

    print("Received date: " .. dateObj:fmt("%Y-%m-%d"))
end)

-- Load date from character
function PLUGIN:GetLastSeen(character)
    local lastSeen = character:GetData("lastSeen")

    if lastSeen then
        local date = ix.date.Construct(lastSeen)
        return ix.date.GetFormatted("%B %d at %I:%M %p", date)
    end

    return "Never"
end

-- Create specific date
function PLUGIN:CreateBirthday(year, month, day)
    local birthdayTable = {
        year = year,
        month = month,
        day = day,
        hour = 0,
        min = 0,
        sec = 0
    }

    return ix.date.Construct(birthdayTable)
end

-- Calculate age
function PLUGIN:GetAge(character)
    local birthData = character:GetData("birthDate")

    if birthData then
        local birthDate = ix.date.Construct(birthData)
        local currentDate = ix.date.Get()
        local age = currentDate:getyear() - birthDate:getyear()

        -- Adjust if birthday hasn't occurred this year
        if currentDate:getmonth() < birthDate:getmonth() or
           (currentDate:getmonth() == birthDate:getmonth() and
            currentDate:getday() < birthDate:getday()) then
            age = age - 1
        end

        return age
    end

    return nil
end
```

### Date Object Methods

**Reference**: https://github.com/Tieske/date (third-party library)

The date object returned by `ix.date.Get()` and `ix.date.Construct()` has many built-in methods:

**Complete Example**:
```lua
local date = ix.date.Get()

-- Getting components
local year = date:getyear()      -- 2024
local month = date:getmonth()    -- 1-12
local day = date:getday()        -- 1-31
local hour = date:gethour()      -- 0-23
local minute = date:getminutes() -- 0-59  (note: getminutes, not getminute)
local second = date:getseconds() -- 0-59

-- Setting components
date:setyear(2077)
date:setmonth(12)
date:setday(25)
date:sethour(18)
date:setminutes(30)
date:setseconds(0)

-- Arithmetic (returns new date object)
local tomorrow = date:adddays(1)
local yesterday = date:adddays(-1)
local nextWeek = date:adddays(7)
local nextMonth = date:addmonths(1)
local nextYear = date:addyears(1)
local oneHourLater = date:addhours(1)
local thirtyMinLater = date:addminutes(30)

-- Comparison
local date1 = ix.date.Get()
local date2 = date1:copy():adddays(1)

if date2 > date1 then
    print("date2 is later")
end

-- Difference calculation
local daysBetween = date2:spandays(date1)  -- Days difference
local hoursBetween = date2:spanhours(date1)  -- Hours difference

-- Day of week
local weekday = date:getweekday()  -- 1=Sunday, 2=Monday, ..., 7=Saturday
local weekdayName = date:fmt("%A")

-- Copy date (important for modifications)
local dateCopy = date:copy()
dateCopy:adddays(5)  -- Original date unchanged
```

## Server-Only Functions

### ix.date.Send (Internal)

**Reference**: `gamemode/core/libs/sh_date.lua:72`

```lua
ix.date.Send(client)
```

Synchronizes current date to clients. Called automatically on player join.

### ix.date.Save (Internal)

**Reference**: `gamemode/core/libs/sh_date.lua:89`

```lua
ix.date.Save()
```

Saves current date to disk. Called automatically by the framework.

## Complete Example

### Time-Based Event System

```lua
PLUGIN.name = "Timed Events"
PLUGIN.description = "Events that occur at specific dates/times"

if (SERVER) then
    PLUGIN.events = PLUGIN.events or {}

    -- Register a timed event
    function PLUGIN:RegisterEvent(name, date, callback)
        self.events[name] = {
            date = ix.date.GetSerialized(date),
            callback = callback
        }
    end

    -- Check events every minute
    function PLUGIN:Think()
        self.nextCheck = self.nextCheck or 0

        if CurTime() >= self.nextCheck then
            self.nextCheck = CurTime() + 60  -- Check every minute

            local currentDate = ix.date.Get()

            for name, event in pairs(self.events) do
                local eventDate = ix.date.Construct(event.date)

                -- Event should trigger if current time >= event time
                if currentDate >= eventDate then
                    print("Triggering event: " .. name)
                    event.callback()
                    self.events[name] = nil  -- Remove one-time event
                end
            end
        end
    end

    -- Example: Schedule event for 7 in-game days from now
    function PLUGIN:ScheduleRaid()
        local raidDate = ix.date.Get():adddays(7):sethour(20)  -- 7 days, 8 PM

        self:RegisterEvent("Raid", raidDate, function()
            for _, client in ipairs(player.GetAll()) do
                client:Notify("The raid is starting!")
            end
        end)

        local dateStr = ix.date.GetFormatted("%B %d at %I:%M %p", raidDate)
        print("Raid scheduled for: " .. dateStr)
    end
end

-- Display in-game date/time on HUD
if (CLIENT) then
    hook.Add("HUDPaint", "ShowDateTime", function()
        local dateStr = ix.date.GetFormatted("%A, %B %d, %Y")
        local timeStr = ix.date.GetFormatted("%I:%M %p")

        draw.SimpleText(dateStr, "DermaDefault", 20, 20, color_white)
        draw.SimpleText(timeStr, "DermaDefault", 20, 35, color_white)
    end)
end
```

## Best Practices

### ✅ DO

- Use `ix.date.Get()` for all in-game date/time operations
- Use `ix.date.GetFormatted()` for displaying dates to players
- Serialize dates with `ix.date.GetSerialized()` before storing/networking
- Use `:copy()` when modifying dates to avoid changing originals
- Configure time scale through config system
- Format dates consistently throughout your schema
- Use date arithmetic methods (`:adddays()`, `:addmonths()`, etc.)

### ❌ DON'T

- Don't use `os.time()` or `os.date()` for in-game dates
- Don't modify `ix.date.current` directly—use config system
- Don't forget to copy dates before modifying: `date:copy():adddays(1)`
- Don't network raw date objects—serialize them first
- Don't assume real-world time matches game time
- Don't hardcode time scales—use `ix.date.timeScale`
- Don't compare date objects with `==` (use `>`, `<`, `>=`, `<=`)

## Common Patterns

### Pattern 1: Character Birthday System

```lua
-- Set character birthday on creation
function PLUGIN:OnCharacterCreated(client, character)
    local birthYear = ix.date.Get():getyear() - character:GetAge()

    local birthDate = ix.date.Construct({
        year = birthYear,
        month = math.random(1, 12),
        day = math.random(1, 28),
        hour = 0,
        min = 0,
        sec = 0
    })

    character:SetData("birthDate", ix.date.GetSerialized(birthDate))
end

-- Display birthday
function PLUGIN:GetBirthday(character)
    local birthData = character:GetData("birthDate")

    if birthData then
        local birthDate = ix.date.Construct(birthData)
        return ix.date.GetFormatted("%B %d, %Y", birthDate)
    end

    return "Unknown"
end
```

### Pattern 2: Cooldown System

```lua
-- Set cooldown
function PLUGIN:SetCooldown(client, action, hours)
    local cooldownDate = ix.date.Get():addhours(hours)
    client:SetData(action .. "Cooldown", ix.date.GetSerialized(cooldownDate))
end

-- Check if cooldown expired
function PLUGIN:CanPerformAction(client, action)
    local cooldownData = client:GetData(action .. "Cooldown")

    if cooldownData then
        local cooldownDate = ix.date.Construct(cooldownData)
        local currentDate = ix.date.Get()

        if currentDate < cooldownDate then
            local minutesLeft = cooldownDate:spanminutes(currentDate)
            return false, "Wait " .. math.ceil(minutesLeft) .. " more minutes"
        end
    end

    return true
end
```

### Pattern 3: Scheduled Announcements

```lua
-- Announce time periodically
function PLUGIN:AnnounceTime()
    timer.Create("TimeAnnouncement", 1800, 0, function()  -- Every 30 real minutes
        local dateStr = ix.date.GetFormatted("%A, %B %d, %Y at %I:%M %p")

        for _, client in ipairs(player.GetAll()) do
            client:ChatPrint("Current date: " .. dateStr)
        end
    end)
end
```

### Pattern 4: Historical Events

```lua
-- Define historical events
PLUGIN.historicalEvents = {
    {
        name = "The Uprising",
        date = {year = 2020, month = 7, day = 4, hour = 0, min = 0, sec = 0},
        description = "Citizens revolted against the Combine."
    }
}

-- Calculate time since event
function PLUGIN:TimeSinceEvent(eventName)
    for _, event in ipairs(self.historicalEvents) do
        if event.name == eventName then
            local eventDate = ix.date.Construct(event.date)
            local currentDate = ix.date.Get()
            local daysSince = currentDate:spandays(eventDate)

            return math.floor(daysSince), event.description
        end
    end
end
```

## Common Issues

### Issue: Time Jumps After Server Restart

**Cause**: Time offset not saved properly
**Fix**: Framework handles this automatically, but ensure date saves on shutdown

```lua
-- Date is automatically saved by framework
-- Check config values are correct
print("Year:", ix.config.Get("year"))
print("Month:", ix.config.Get("month"))
print("Day:", ix.config.Get("day"))
```

### Issue: Date Comparison Doesn't Work

**Cause**: Comparing date objects with `==`
**Fix**: Use comparison operators `>`, `<`, `>=`, `<=`

```lua
-- WRONG
if date1 == date2 then  -- Don't use ==

-- CORRECT
if date1 >= date2 then  -- Use >=, <=, >, <
```

### Issue: Modifying Date Changes Original

**Cause**: Date objects are references, not copies
**Fix**: Use `:copy()` before modifying

```lua
-- WRONG
local date1 = ix.date.Get()
local date2 = date1  -- Same reference!
date2:adddays(1)  -- Modifies date1 too!

-- CORRECT
local date1 = ix.date.Get()
local date2 = date1:copy()  -- New copy
date2:adddays(1)  -- date1 unchanged
```

## See Also

- [Configuration System](../systems/configuration.md) - Time scale and date configuration
- [Data System](../libraries/data.md) - Persisting dates
- [Character System](../systems/character.md) - Character-specific dates
- Third-party Date Library: https://github.com/Tieske/date
- Source: `gamemode/core/libs/sh_date.lua`
