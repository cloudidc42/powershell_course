# Part 24: Template Engine และ HTML Generation

> **ระดับ**: 🟠 Advanced | **เวลา**: ~4 ชั่วขนึง

---

## 1. Simple Template Engine

```powershell
function Invoke-Template {
    param(
        [string]$Template,
        [hashtable]$Data
    )
    
    $result = $Template
    
    # {{variable}} substitution
    foreach ($key in $Data.Keys) {
        $result = $result -replace "\{\{\s*$key\s*\}\}", [regex]::Escape($Data[$key].ToString())
    }
    
    # {{#each items}}...{{/each}}
    $result = [regex]::Replace($result, 
        '\{\{#each (\w+)\}\}([\s\S]*?)\{\{/each\}\}',
        {
            param($m)
            $varName = $m.Groups[1].Value
            $body    = $m.Groups[2].Value
            $items   = $Data[$varName]
            if (!$items) { return '' }
            
            ($items | ForEach-Object {
                $item = $_
                $block = $body
                # Replace {{this}} or {{property}}
                if ($item -is [hashtable] -or $item -is [PSObject]) {
                    $props = if ($item -is [hashtable]) { $item }
                             else { $item.PSObject.Properties | ForEach-Object -Begin { @{} } -Process { $_[0][$_.Name] = $_.Value } -End { $_[0] } }
                    foreach ($k in $props.Keys) {
                        $block = $block -replace "\{\{\s*$k\s*\}\}", $props[$k]
                    }
                } else {
                    $block = $block -replace '\{\{this\}\}', $item
                }
                $block
            }) -join ''
        }
    )
    
    # {{#if condition}}...{{/if}}
    $result = [regex]::Replace($result,
        '\{\{#if (\w+)\}\}([\s\S]*?)\{\{/if\}\}',
        {
            param($m)
            $varName = $m.Groups[1].Value
            $body    = $m.Groups[2].Value
            if ($Data[$varName]) { $body } else { '' }
        }
    )
    
    return $result
}

# Usage
$template = @'
<!DOCTYPE html>
<html>
<head><title>{{title}}</title></head>
<body>
  <h1>{{heading}}</h1>
  {{#if showWelcome}}
  <p>Welcome, {{user}}!</p>
  {{/if}}
  <ul>
  {{#each items}}
    <li>{{name}} - ${{price}}</li>
  {{/each}}
  </ul>
</body>
</html>
'@

$html = Invoke-Template $template @{
    title      = 'My Store'
    heading    = 'Products'
    showWelcome = $true
    user       = 'Alice'
    items      = @(
        @{name='Widget A'; price='9.99'}
        @{name='Widget B'; price='14.99'}
        @{name='Widget C'; price='4.99'}
    )
}

Write-Host $html
```

---

## 2. HTML Helpers

```powershell
class HtmlBuilder {
    hidden [System.Text.StringBuilder]$Sb
    
    HtmlBuilder() {
        $this.Sb = [System.Text.StringBuilder]::new()
    }
    
    static [string] Encode([string]$text) {
        [System.Net.WebUtility]::HtmlEncode($text)
    }
    
    [HtmlBuilder] Append([string]$html) {
        $this.Sb.Append($html) | Out-Null
        return $this
    }
    
    [HtmlBuilder] Tag([string]$tag, [string]$content, [hashtable]$attrs = @{}) {
        $attrStr = ($attrs.GetEnumerator() | ForEach-Object { " $($_.Key)=`"$($_.Value)`"" }) -join ''
        $this.Sb.Append("<$tag$attrStr>$content</$tag>") | Out-Null
        return $this
    }
    
    [HtmlBuilder] Table([array]$data, [string[]]$columns = @()) {
        if ($data.Count -eq 0) {
            $this.Sb.Append('<p>No data</p>') | Out-Null
            return $this
        }
        
        if ($columns.Count -eq 0) {
            $columns = $data[0].PSObject.Properties.Name
        }
        
        $this.Sb.Append('<table class="table"><thead><tr>') | Out-Null
        $columns | ForEach-Object { $this.Sb.Append("<th>$_</th>") | Out-Null }
        $this.Sb.Append('</tr></thead><tbody>') | Out-Null
        
        foreach ($row in $data) {
            $this.Sb.Append('<tr>') | Out-Null
            foreach ($col in $columns) {
                $val = [HtmlBuilder]::Encode($row.$col?.ToString() ?? '')
                $this.Sb.Append("<td>$val</td>") | Out-Null
            }
            $this.Sb.Append('</tr>') | Out-Null
        }
        
        $this.Sb.Append('</tbody></table>') | Out-Null
        return $this
    }
    
    [string] ToString() { return $this.Sb.ToString() }
}

# Usage
$html = [HtmlBuilder]::new()
$html.Tag('h1', 'System Report', @{class='title'})
     .Tag('p', "Generated: $(Get-Date)")
     .Table((Get-Process | Select-Object Name, CPU, WorkingSet -First 10),
            @('Name', 'CPU', 'WorkingSet'))

Write-Host $html.ToString()
```

---

## 3. Static File Server

```powershell
function Invoke-StaticFile {
    param([HttpContext]$Ctx, [string]$RootPath = '.')
    
    $path = $Ctx.Request.Url.AbsolutePath.TrimStart('/')
    if (!$path) { $path = 'index.html' }
    
    $fullPath = Join-Path $RootPath $path
    # Prevent path traversal!
    $fullPath = [System.IO.Path]::GetFullPath($fullPath)
    $rootFull = [System.IO.Path]::GetFullPath($RootPath)
    
    if (!$fullPath.StartsWith($rootFull)) {
        $Ctx.Json(@{error='Forbidden'}, 403)
        return
    }
    
    if (!(Test-Path $fullPath -PathType Leaf)) {
        $Ctx.NotFound("File not found: $path")
        return
    }
    
    $ext = [System.IO.Path]::GetExtension($fullPath).ToLower()
    $contentType = switch ($ext) {
        '.html' { 'text/html; charset=utf-8' }
        '.css'  { 'text/css' }
        '.js'   { 'application/javascript' }
        '.json' { 'application/json' }
        '.png'  { 'image/png' }
        '.jpg'  { 'image/jpeg' }
        '.gif'  { 'image/gif' }
        '.svg'  { 'image/svg+xml' }
        '.ico'  { 'image/x-icon' }
        '.woff2'{ 'font/woff2' }
        default { 'application/octet-stream' }
    }
    
    $bytes = [System.IO.File]::ReadAllBytes($fullPath)
    $Ctx.Response.ContentType    = $contentType
    $Ctx.Response.ContentLength64 = $bytes.Length
    $Ctx.Response.StatusCode     = 200
    
    # Cache-Control for static assets
    if ($ext -in @('.png','.jpg','.gif','.svg','.ico','.woff2','.css','.js')) {
        $Ctx.Response.AddHeader('Cache-Control', 'public, max-age=86400')
    }
    
    $Ctx.Response.OutputStream.Write($bytes, 0, $bytes.Length)
    $Ctx.Response.OutputStream.Close()
}
```

---

**ก่อนหน้า ← [Part 23](Part-23.md) | ต่อไป → [Part 25: Full Stack App](Part-25.md)**
