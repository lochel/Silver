#meta #lib/silver

This file implements a **journal calendar system**.
${journal.calendar(date.today(), "Journal/")}

**Config**
* `journal.weekNumbers`: Toggle week number display in the calendar table.
* `journal.template`: Default content for new daily journal pages.


## Implementation

```space-lua
-- priority: 9
journal = journal or {}
journal.weekNumbers = true
journal.template = "## Schedule\n* \n\n## Notes\n* "

function journal.calendar(root, today)
  local daysOfWeek = {"Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"}

  local function getWeekday(y, m, d)
    local t = os.time({ year = y, month = m, day = d })
    local w = tonumber(os.date("%w", t))
    return (w + 6) % 7
  end

  local function daysInMonth(y, m)
    if m == 2 then
      if (y % 4 == 0 and y % 100 ~= 0) or (y % 400 == 0) then
        return 29
      else
        return 28
      end
    elseif m == 4 or m == 6 or m == 9 or m == 11 then
      return 30
    else
      return 31
    end
  end

  local function getWeekNumber(y, m, d)
    local t = os.time({ year = y, month = m, day = d })
    local wn = tonumber(os.date("%V", t))
    return string.format("[[" .. root .. "%04d-v%2d|v%2d]]", y, wn, wn)
  end

  local year, month, _ = date.fromString(today)
  local markdown = ""

  if journal.weekNumbers then
    markdown = markdown .. "| #  | " .. table.concat(daysOfWeek, " | ") .. " |\n"
    markdown = markdown .. "|----|" .. string.rep("----|", 7) .. "\n"
  else
    markdown = markdown .. "| " .. table.concat(daysOfWeek, " | ") .. " |\n"
    markdown = markdown .. string.rep("|----", 7) .. "|\n"
  end

  local startDay = getWeekday(year, month, 1)
  local day = 1
  local row = ""

  if journal.weekNumbers then
    row = row .. "| " .. getWeekNumber(year, month, 1) .. " "
  end
  row = row .. string.rep("|    ", startDay)

  while day <= daysInMonth(year, month) do
    local dateStr = date.toString(year, month, day)
    local datePage = root .. dateStr
    local dayStr = string.format("[[%s|%02d]]", datePage, day)

    if dateStr == date.today() then
      dayStr = string.format("[[%s|🔴]]", datePage)
    elseif dateStr == today then
      dayStr = string.format("[[%s|⭕]]", datePage)
    elseif space.fileExists(datePage .. ".md") then
      dayStr = "**" .. dayStr .. "**"
    else
      dayStr = "*" .. dayStr .. "*"
    end

    row = row .. "| " .. dayStr .. " "

    local weekday = getWeekday(year, month, day)
    if weekday == 6 then
      row = row .. "|\n"
      markdown = markdown .. row
      day = day + 1
      if day <= daysInMonth(year, month) then
        row = ""
        if journal.weekNumbers then
          row = row .. "| " .. getWeekNumber(year, month, day) .. " "
        end
      end
    else
      day = day + 1
    end
  end

  local lastWeekday = getWeekday(year, month, daysInMonth(year, month))
  if lastWeekday < 6 then
    row = row .. string.rep("|    ", 6 - lastWeekday) .. "|\n"
    markdown = markdown .. row
  end

  return markdown
end

function journal.summary(root, days)
  local year, month, day = date.fromString(date.today())
  local text = ""

  for i = 0, days do
    local ts = os.time({year=year, month=month, day=day}) + i * 86400
    local dateStr = os.date("%Y-%m-%d", ts)
    if space.fileExists(root .. dateStr .. ".md") then
      text = text .. "## [[" .. root .. dateStr .. "|" .. os.date("%A, %d/%m", ts) .. "]]\n"
      text = text .. "![[" .. root .. dateStr .. "|" .. dateStr .. "]]\n"
    end
  end

  return text
end
```


## Widget
```space-lua
widgets = widgets or {}

function widgets.journalCalendar(options)
  options = options or {}

  local pageName = editor.getCurrentPage()
  local root, ymd = pageName:match("^(.*/)(%d+%-%d+%-%d+)$")

  if root == nil then
    return
  end

  local markdown = journal.calendar(root, ymd)

  return widget.new {
    markdown = markdown
  }
end

-- Register widget to render at the top of the page
event.listen {
  name = "hooks:renderTopWidgets",
  run = function(e)
    return widgets.journalCalendar()
  end
}
```


## Templates
```space-lua
event.listen {
  name = "editor:pageCreating",
  run = function(e)
    local root, y, m, d = e.data.name:match("^(.*/)(%d+)%-(%d+)%-(%d+)$")
    if root == nil then
      return
    end


    return {
      text = journal.template,
      perm = "rw"
    }
  end
}
```

```space-lua
event.listen {
  name = "editor:pageCreating",
  run = function(e)
    local root = e.data.name:match("^(.*/)-Journal$")
    if root == nil then
      return
    end

    local text = "## Journal Pages\n${query[[from index.tag \"page\" where name:startsWith(\"" .. e.data.name .. "/\") order by name desc select \"[[\" .. name .. \"]]\"]]}\n"

    return {
      text = text,
      perm = "ro"
    }
  end
}
```

```space-lua
event.listen {
  name = "editor:pageCreating",
  run = function(e)
    local root, y, wn = e.data.name:match("^(.*/)(%d+)%-(v%d+)$")
    if not wn then return end

    local year = tonumber(y)
    local week = tonumber(wn:sub(2))  -- remove "v"

    local approx = os.time({year = year, month = 1, day = 1}) + (week - 2) * 7 * 86400

    local text = "# " .. year .. " " .. wn .. "\n\n"
    for i = 0, 13 do  -- check two full weeks (14 days)
      local ts = approx + i * 86400
      local w = tonumber(os.date("%V", ts))
      if w == week then
        local dateStr = os.date("%Y-%m-%d", ts)
        if space.fileExists(root .. dateStr .. ".md") then
          text = text .. "## [[" .. root .. dateStr .. "|" .. os.date("%A, %d/%m", ts) .. "]]\n"
          text = text .. "![[" .. root .. dateStr .. "|" .. dateStr .. "]]\n\n"
        end
      end
    end

    return {
      text = text,
      perm = "ro"
    }
  end
}
```
