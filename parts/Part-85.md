# Part 85: Plugin Architecture และ Extensibility

> **ระดับ**: 🟠 World-Class | **เวลา**: ~5 ชั่วโมง

---

## 1. Plugin System

```powershell
# Plugin registry and loader
class PluginRegistry {
    hidden [hashtable]$_plugins     = @{}
    hidden [hashtable]$_metadata    = @{}
    hidden [string]$_pluginDir
    
    PluginRegistry([string]$pluginDir) {
        $this._pluginDir = $pluginDir
    }
    
    [void] DiscoverAndLoad() {
        $manifests = Get-ChildItem $this._pluginDir -Filter 'plugin.json' -Recurse
        
        foreach ($manifest in $manifests) {
            try {
                $meta    = Get-Content $manifest.FullName | ConvertFrom-Json
                $entryPt = Join-Path $manifest.Directory $meta.entry
                
                if (-not (Test-Path $entryPt)) {
                    Write-Warning "Plugin entry not found: $entryPt"
                    continue
                }
                
                # Load plugin in isolated runspace
                $rs = [runspacefactory]::CreateRunspace()
                $rs.Open()
                
                $ps = [powershell]::Create()
                $ps.Runspace = $rs
                $ps.AddScript("Set-StrictMode -Version Latest; . '$entryPt'; Export-ModuleMember *") | Out-Null
                $ps.Invoke()
                
                $this._plugins[$meta.name]  = $ps
                $this._metadata[$meta.name] = $meta
                
                Write-Host "Loaded plugin: $($meta.name) v$($meta.version)" -ForegroundColor Green
            } catch {
                Write-Warning "Failed to load plugin $($manifest.Directory.Name): $_"
            }
        }
    }
    
    [object] InvokePlugin([string]$name, [string]$method, [object[]]$args = @()) {
        $ps = $this._plugins[$name]
        if (-not $ps) { throw "Plugin not found: $name" }
        
        $ps.Commands.Clear()
        $ps.AddCommand($method) | Out-Null
        foreach ($arg in $args) { $ps.AddArgument($arg) | Out-Null }
        
        return $ps.Invoke()
    }
    
    [hashtable[]] List() {
        return $this._metadata.Values | ForEach-Object {
            @{ Name=$_.name; Version=$_.version; Description=$_.description }
        }
    }
    
    [void] Unload([string]$name) {
        $ps = $this._plugins[$name]
        if ($ps) {
            $ps.Runspace.Dispose()
            $ps.Dispose()
            $this._plugins.Remove($name)
            $this._metadata.Remove($name)
        }
    }
}

# Example plugin.json:
# {
#   "name": "slack-notifier",
#   "version": "1.0.0",
#   "description": "Sends notifications to Slack",
#   "entry": "slack-notifier.ps1",
#   "hooks": ["OnDeploy", "OnAlert"]
# }

$registry = [PluginRegistry]::new('./plugins')
$registry.DiscoverAndLoad()
$registry.List() | Format-Table
$registry.InvokePlugin('slack-notifier', 'Send-SlackMessage', @('Deploy complete!', '#deploys'))
```

---

## 2. Hook System

```powershell
# Event hooks for extensibility
class HookSystem {
    hidden [hashtable]$_hooks = @{}
    
    [void] Register([string]$hookName, [string]$id, [scriptblock]$handler, [int]$priority = 10) {
        if (-not $this._hooks[$hookName]) { $this._hooks[$hookName] = @() }
        $this._hooks[$hookName] += @{ id=$id; handler=$handler; priority=$priority }
        $this._hooks[$hookName] = @($this._hooks[$hookName] | Sort-Object { $_['priority'] })
    }
    
    [void] Unregister([string]$hookName, [string]$id) {
        if ($this._hooks[$hookName]) {
            $this._hooks[$hookName] = $this._hooks[$hookName] | Where-Object { $_['id'] -ne $id }
        }
    }
    
    [object] Invoke([string]$hookName, [hashtable]$context) {
        $handlers = $this._hooks[$hookName]
        if (-not $handlers) { return $context }
        
        foreach ($h in $handlers) {
            try {
                $result = & $h['handler'] $context
                # Handlers can modify context
                if ($result -is [hashtable]) { $context = $context + $result }
                # Stop if handler returns $false
                if ($result -eq $false) {
                    Write-Verbose "Hook chain stopped by $($h['id'])"
                    break
                }
            } catch {
                Write-Warning "Hook $hookName/$($h['id']) failed: $_"
            }
        }
        return $context
    }
}

$hooks = [HookSystem]::new()

# Register hooks
$hooks.Register('before:deploy', 'security-check', {
    param($ctx)
    Write-Host "[Hook] Security check for $($ctx.app)" -ForegroundColor Yellow
    if ($ctx.env -eq 'production' -and -not $ctx.approved) {
        throw "Production deploy requires approval"
    }
}, priority=1)

$hooks.Register('before:deploy', 'notify-slack', {
    param($ctx)
    Write-Host "[Hook] Notifying Slack: Deploying $($ctx.app) v$($ctx.version)" -ForegroundColor Cyan
}, priority=5)

$hooks.Register('after:deploy', 'invalidate-cache', {
    param($ctx)
    Write-Host "[Hook] Invalidating CDN cache for $($ctx.app)" -ForegroundColor Green
}, priority=10)

# Invoke hooks
$deployCtx = @{ app='myapp'; version='2.1.0'; env='production'; approved=$true }
$deployCtx = $hooks.Invoke('before:deploy', $deployCtx)
# ... do deployment ...
$hooks.Invoke('after:deploy', $deployCtx)
```

---

## 3. Dynamic Module Loading

```powershell
# Hot-reload module system
class ModuleWatcher {
    hidden [System.IO.FileSystemWatcher]$_watcher
    hidden [hashtable]$_loadedModules = @{}
    
    ModuleWatcher([string]$modulesPath) {
        $this._watcher = [System.IO.FileSystemWatcher]::new($modulesPath, '*.ps1')
        $this._watcher.IncludeSubdirectories = $false
        $this._watcher.NotifyFilter = [System.IO.NotifyFilters]'LastWrite'
        
        $watcher = $this  # capture reference
        $this._watcher.Add_Changed({
            param($s, $e)
            Write-Host "[Reload] Detected change: $($e.Name)" -ForegroundColor Yellow
            $watcher.ReloadModule($e.FullPath)
        })
        $this._watcher.EnableRaisingEvents = $true
    }
    
    [void] LoadAll() {
        Get-ChildItem $this._watcher.Path -Filter '*.ps1' | ForEach-Object {
            $this.LoadModule($_.FullName)
        }
    }
    
    [void] LoadModule([string]$path) {
        Write-Host "Loading: $(Split-Path $path -Leaf)" -ForegroundColor Green
        . $path
        $this._loadedModules[$path] = [datetime]::Now
    }
    
    [void] ReloadModule([string]$path) {
        $this.LoadModule($path)
        Write-Host "Reloaded: $(Split-Path $path -Leaf)" -ForegroundColor Cyan
    }
    
    [void] Stop() { $this._watcher.EnableRaisingEvents = $false; $this._watcher.Dispose() }
}

$moduleWatcher = [ModuleWatcher]::new('./hot-modules')
$moduleWatcher.LoadAll()

Write-Host 'Module watcher active. Edit .ps1 files to hot-reload.' -ForegroundColor Cyan
# Keep running...
try {
    while ($true) { Start-Sleep 1 }
} finally {
    $moduleWatcher.Stop()
}
```

---

**ก่อนหน้า ← [Part 84](Part-84.md) | ต่อไป → [Part 86: Windows Admin Automation](Part-86.md)**
