#meta #lib/silver

```space-lua
-- priority: 10

date = date or {}

function date.fromString(str)
  local y, m, d = str:match("(%d+)%-(%d+)%-(%d+)")
  return tonumber(y), tonumber(m), tonumber(d)
end

function date.toString(y, m, d)
  return string.format("%04d-%02d-%02d", y, m, d)
end

function date.niceDate(iso_timestamp)
  local year, month, day, hour, min, sec = iso_timestamp:match("(%d+)%-(%d+)%-(%d+)T(%d+):(%d+):(%d+)")

  if not year then
    return "Invalid ISO timestamp"
  end

  -- Create a Lua table and convert to a timestamp
  local time_table = {
    year = tonumber(year),
    month = tonumber(month),
    day = tonumber(day),
    hour = tonumber(hour),
    min = tonumber(min),
    sec = tonumber(sec),
    isdst = false, -- assumes UTC
  }

  local timestamp = os.time(time_table)

  return os.date("%Y-%m-%d %H:%M", timestamp)
end
```
