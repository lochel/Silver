#meta #lib/silver

These Silver modules are currently installed:
${silver.list()}

## Config

In your [[^CONFIG]] page (or anywhere else), configure the library repository (e.g., your fork of the official Silver repo) and specify which modules to install.
By default, the official Silver repository is used, and all modules are selected.

### `silver.repo`

GitHub repository to fetch modules from (e.g., ${silver.repo})

### `silver.modules`

Modules to install: ${"{" .. table.concat(silver.modules, ", ") .. "}"}

### `silver.path`

Local path where modules are installed (e.g., ${silver.path})

## Commands

```space-lua
command.define {
  name = "Silver: Upgrade",
  run = function()
    silver.downloadModule()
    for _, module in ipairs(silver.modules) do
      silver.downloadModule(module)
    end
  end
}
```

## Implementation

```space-lua
-- priority: 100

silver = silver or {}
silver.path = "Library/Silver"
silver.repo = "lochel/Silver"
silver.modules = {}

function silver.list()
  local modules = query[[
    from index.tag "page"
    where name:startsWith(silver.path .. "/")
  ]]
  local result = ""
  for _, item in ipairs(modules) do
    result = result .. "* [[" .. item.name .. "]]\n"
  end
  return result
end

function silver.isLibPage(module)
  local pages = query[[
    from index.tag "page"
    where name == silver.getPath(module) and table.includes(tags, "lib/silver")
  ]]
  return #pages == 1
end

function silver.getPath(module)
  if not module then
    return silver.path
  else
    return silver.path .. "/" .. module
  end
end

function silver.getRemotePath(module)
  if not module then
    return "https://api.github.com/repos/" .. silver.repo .. "/contents/Silver.md"
  else
    return "https://api.github.com/repos/" .. silver.repo .. "/contents/Silver/" .. module .. ".md"
  end
end

function silver.downloadModule(module)
  local localPath = silver.getPath(module)
  local remotePath = silver.getRemotePath(module)

  -- Get metadata from GitHub
  local resp = http.request(remotePath, {
    headers = { Accept = "application/vnd.github.v3+json" }
  })

  if not resp or not resp.ok or not resp.body then
    editor.flashNotification("Failed to fetch metadata for " .. localPath, "error")
    return false
  end

  local downloadUrl = resp.body.download_url
  if not downloadUrl then
    editor.flashNotification("No download URL for module: " .. localPath, "error")
    return false
  end

  -- Download actual content
  local contentResp = http.request(downloadUrl)
  if not contentResp or not contentResp.ok or not contentResp.body then
    editor.flashNotification("Failed to download module content: " .. localPath, "error")
    return false
  end

  local content = contentResp.body

  -- Avoid overwriting existing non-library pages
  if space.fileExists(localPath .. ".md") and not silver.isLibPage(module) then
    editor.flashNotification("Page exists, not overwriting: " .. localPath, "error")
    return false
  end

  space.writePage(localPath, content)
  editor.flashNotification("Module installed: " .. localPath)
  return true
end
```
