# Date API (ix.date)

> **Reference**: `gamemode/core/libs/sh_date.lua`

The date API provides persistent in-game date and time handling independent of the Unix epoch (pre-1970 dates). It uses a third-party date library for advanced date manipulation and formatting.

## ⚠️ Important: Use Built-in Helix Date System

**Always use Helix's built-in date functions** rather than `os.time()` or `os.date()` for in-game time. The framework automatically provides:
- Dates before 1970 (not limited by Unix epoch)
- Configurable time scale (real seconds per in-game minute)
- Automatic persistence and synchronization
- Advanced date manipulation via third-party library
- Automatic config integration
- Map-specific date tracking

## Core Concepts

### What is the Date System?

The date system provides roleplay-friendly time tracking:
- Independent from real-world time
- Configurable time scale (e.g., 1 real second = 1 in-game minute)
- Persists across server restarts
- Can represent any date (including historical/future dates)
- Automatically syncs to clients
- Uses third-party date library for advanced features

### Key Terms

**Date Object**: Special table with date manipulation methods
**Time Scale**: Real seconds per in-game minute (default: 60 = real-time)
**Start Time**: Reference point for calculating current in-game date
**Serialized Date**: Date saved as plain table for storage/networking

## Library Tables

### ix.date.current

**Reference**: `gamemode/core/libs/sh_date.lua:18`

**Realm**: Shared

Current in-game date object (base reference).

### ix.date.timeScale

**Reference**: `gamemode/core/libs/sh_date.lua:17`

**Realm**: Shared

Seconds per in-game minute. Lower = faster time.

## Library Functions

### ix.date.Get

**Reference**: `gamemode/core/libs/sh_date.lua:117`

**Realm**: Shared

```lua
local dateObj = ix.date.Get()
```

Returns the current in-game date/time.

**Returns**: (date) Current date object

**Example**:
```lua
-- Get current date
local now = ix.date.Get()

-- Access date components
print(now:getyear())    -- 2024
print(now:getmonth())   -- 11
print(now:getday())     -- 5
print(now:gethours())   -- 14
print(now:getminutes()) -- 30

-- Format as string
print(now:fmt("%Y-%m-%d %H:%M"))  -- "2024-11-05 14:30"
```

**Date Object Methods** (from third-party library):
- `:getyear()` - Year
- `:getmonth()` - Month (1-12)
- `:getday()` - Day of month
- `:gethours()` - Hours (0-23)
- `:getminutes()` - Minutes (0-59)
- `:getseconds()` - Seconds (0-59)
- `:fmt(format)` - Format date string
- `:copy()` - Create copy
- `:adddays(n)` - Add days
- `:addmonths(n)` - Add months
- `:addyears(n)` - Add years
- `:addhours(n)` - Add hours
- `:addminutes(n)` - Add minutes

### ix.date.GetFormatted

**Reference**: `gamemode/core/libs/sh_date.lua:128`

**Realm**: Shared

```lua
local formatted = ix.date.GetFormatted(format, currentDate)
```

Returns formatted date string.

**Parameters**:
- `format` (string) - Format string (see strftime format)
- `currentDate` (date, optional) - Date to format (default: current)

**Returns**: (string) Formatted date

**Example**:
```lua
-- Common formats
print(ix.date.GetFormatted("%Y-%m-%d"))        -- "2024-11-05"
print(ix.date.GetFormatted("%H:%M:%S"))        -- "14:30:00"
print(ix.date.GetFormatted("%A, %B %d, %Y"))  -- "Tuesday, November 05, 2024"
print(ix.date.GetFormatted("%I:%M %p"))        -- "02:30 PM"

-- Custom date
local customDate = ix.date.lib({year = 1942, month = 6, day = 6})
print(ix.date.GetFormatted("%Y-%m-%d", customDate))  -- "1942-06-06"

-- Show in-game time to player
client:ChatPrint("Current time: " .. ix.date.GetFormatted("%H:%M"))
```

**Format Codes**:
- `%Y` - 4-digit year (2024)
- `%m` - 2-digit month (01-12)
- `%d` - 2-digit day (01-31)
- `%H` - 24-hour (00-23)
- `%I` - 12-hour (01-12)
- `%M` - Minutes (00-59)
- `%S` - Seconds (00-59)
- `%p` - AM/PM
- `%A` - Full weekday (Monday)
- `%a` - Short weekday (Mon)
- `%B` - Full month (January)
- `%b` - Short month (Jan)

### ix.date.GetSerialized

**Reference**: `gamemode/core/libs/sh_date.lua:136`

**Realm**: Shared

```lua
local serialized = ix.date.GetSerialized(currentDate)
```

Converts date to table for saving/networking.

**Parameters**:
- `currentDate` (date, optional) - Date to serialize (default: current)

**Returns**: (table) Serialized date

**Example**:
```lua
-- Serialize current date
local serial = ix.date.GetSerialized()
PrintTable(serial)
-- {year = 2024, month = 11, day = 5, hour = 14, min = 30, sec = 0}

-- Save to character
character:SetData("birthdate", ix.date.GetSerialized())

-- Network to client
net.WriteTable(ix.date.GetSerialized())
```

### ix.date.Construct

**Reference**: `gamemode/core/libs/sh_date.lua:144`

**Realm**: Shared

```lua
local dateObj = ix.date.Construct(dateTable)
```

Constructs date object from table.

**Parameters**:
- `dateTable` (table) - Serialized date

**Returns**: (date) Date object

**Example**:
```lua
-- From serialized data
local saved = character:GetData("birthdate")
local birthDate = ix.date.Construct(saved)
print(birthDate:fmt("%Y-%m-%d"))

-- Create custom date
local ww2Start = ix.date.Construct({
    year = 1939,
    month = 9,
    day = 1
})
```

### ix.date.Send

**Reference**: `gamemode/core/libs/sh_date.lua:72`

**Realm**: Server (internal)

```lua
ix.date.Send(client)
```

Synchronizes date to clients.

**Note**: Called automatically by framework.

## Complete Examples

### Time Display HUD

```lua
-- Client-side HUD showing in-game time
hook.Add("HUDPaint", "ShowTime", function()
    local time = ix.date.GetFormatted("%I:%M %p")
    local date = ix.date.GetFormatted("%A, %B %d, %Y")

    draw.SimpleText(time, "DermaLarge", ScrW() - 10, 10, color_white, TEXT_ALIGN_RIGHT)
    draw.SimpleText(date, "DermaDefault", ScrW() - 10, 50, color_white, TEXT_ALIGN_RIGHT)
end)
```

### Time-Based Events

```lua
-- Check if it's nighttime
function IsNighttime()
    local hour = ix.date.Get():gethours()
    return hour >= 20 or hour < 6
end

-- Spawn more dangerous NPCs at night
hook.Add("Think", "NightSpawns", function()
    if IsNighttime() and math.random(1, 1000) == 1 then
        -- Spawn night creatures
    end
end)
```

### Age Calculation

```lua
-- Calculate character age from birthdate
function GetCharacterAge(character)
    local birthdate = character:GetData("birthdate")
    if not birthdate then return nil end

    local birth = ix.date.Construct(birthdate)
    local now = ix.date.Get()

    local age = now:getyear() - birth:getyear()

    -- Adjust if birthday hasn't occurred this year
    if now:getmonth() < birth:getmonth() or
       (now:getmonth() == birth:getmonth() and now:getday() < birth:getday()) then
        age = age - 1
    end

    return age
end

-- Set birthdate on character creation
hook.Add("OnCharacterCreated", "SetBirthdate", function(client, character)
    character:SetData("birthdate", ix.date.GetSerialized())
end)
```

### Scheduled Events

```lua
-- Event that happens at specific in-game time
local nextEventDate = ix.date.Construct({
    year = 2024,
    month = 12,
    day = 25,
    hour = 12,
    min = 0
})

hook.Add("Think", "CheckEvent", function()
    local now = ix.date.Get()

    if now >= nextEventDate then
        -- Trigger event
        ix.util.Notify("Holiday event starting!")

        -- Schedule next occurrence
        nextEventDate = nextEventDate:copy():addyears(1)
    end
end)
```

### Time Command

```lua
ix.command.Add("Time", {
    description = "Shows current in-game date and time",
    OnRun = function(self, client)
        local time = ix.date.GetFormatted("%I:%M %p")
        local date = ix.date.GetFormatted("%A, %B %d, %Y")

        client:ChatPrint("Time: " .. time)
        client:ChatPrint("Date: " .. date)
    end
})
```

### Day/Night Cycle

```lua
-- Change lighting based on time
timer.Create("DayNightCycle", 60, 0, function()
    local hour = ix.date.Get():gethours()

    if hour >= 6 and hour < 20 then
        -- Daytime lighting
        RunConsoleCommand("pp_colour_brightness", "0")
    else
        -- Nighttime lighting
        RunConsoleCommand("pp_colour_brightness", "-0.02")
        RunConsoleCommand("pp_colour_colour", "0.8")
    end
end)
```

## Best Practices

### ✅ DO

- Use ix.date.Get() for current in-game time
- Store dates with GetSerialized() and restore with Construct()
- Use GetFormatted() for display strings
- Check date documentation at https://github.com/Tieske/date
- Set appropriate time scale in config (lower = faster)
- Use date objects for calculations and comparisons

### ❌ DON'T

- Don't use os.time() or os.date() for in-game time
- Don't modify ix.date.current directly
- Don't assume real-time (check timeScale)
- Don't store date objects directly (serialize first)
- Don't forget dates are server-authoritative
- Don't use for real-world timestamps (use os.time for that)

## Common Issues

### Date Not Advancing

**Cause**: Time scale set to very high value or date not initialized.

**Fix**: Check config:
```lua
print(ix.date.timeScale)  -- Should be reasonable (1-60)
```

### Date Resets on Restart

**Cause**: Date not saving properly or data folder issues.

**Fix**: Date saves automatically. Check `data/helix/schema/date.txt`.

### Wrong Date Format

**Cause**: Invalid format string.

**Fix**: Use valid strftime codes:
```lua
-- WRONG
ix.date.GetFormatted("YYYY-MM-DD")

-- CORRECT
ix.date.GetFormatted("%Y-%m-%d")
```

## Related Hooks

Hook usage with dates:

```lua
-- Use current date in hooks
hook.Add("PlayerSay", "TimeCheck", function(client, text)
    if text == "/time" then
        local time = ix.date.GetFormatted("%H:%M")
        client:ChatPrint("Current time: " .. time)
        return ""
    end
end)
```

## See Also

- [Config API](config.md) - Setting year, month, day, secondsPerMinute
- [Data API](data.md) - Date persistence
- Third-party date library: https://github.com/Tieske/date
- Source: `gamemode/core/libs/sh_date.lua`
