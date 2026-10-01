# Part 73: Web Scraping ด้วย PowerShell

> **ระดับ**: 🟡 Professional | **เวลา**: ~4 ชั่วโมง

---

## 1. Basic Web Scraping

```powershell
# Scrape with Invoke-WebRequest
function Get-WebPage {
    param([string]$Url, [hashtable]$Headers = @{})
    
    $defaultHeaders = @{
        'User-Agent' = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
        'Accept'     = 'text/html,application/xhtml+xml'
    }
    
    return Invoke-WebRequest -Uri $Url `
        -Headers ($defaultHeaders + $Headers) `
        -UseBasicParsing
}

# Parse HTML links
function Get-Links {
    param([string]$Url, [string]$Pattern = '.*')
    
    $page = Get-WebPage $Url
    $page.Links | Where-Object { $_.href -match $Pattern } |
        Select-Object @{n='Text';e={$_.innerText.Trim()}}, @{n='URL';e={$_.href}} |
        Where-Object { $_.URL -and $_.URL -ne '#' }
}

# Extract tables from HTML
function Get-HtmlTable {
    param([string]$Url, [int]$TableIndex = 0)
    
    $page   = Get-WebPage $Url
    $tables = @($page.ParsedHtml.getElementsByTagName('table'))
    
    if ($tables.Count -le $TableIndex) {
        throw "Table index $TableIndex not found. Found $($tables.Count) tables."
    }
    
    $table   = $tables[$TableIndex]
    $headers = @($table.rows[0].cells | ForEach-Object { $_.innerText.Trim() })
    
    $rows = for ($i = 1; $i -lt $table.rows.length; $i++) {
        $row = $table.rows[$i]
        $obj = [ordered]@{}
        for ($j = 0; $j -lt $headers.Count; $j++) {
            $obj[$headers[$j]] = $row.cells[$j].innerText.Trim()
        }
        [PSCustomObject]$obj
    }
    return $rows
}

# Scrape stock prices
$stockPage = Get-WebPage 'https://finance.example.com/quotes'
$prices    = $stockPage.Content | Select-String -Pattern '"price":\s*([\d.]+)' -AllMatches |
    ForEach-Object { $_.Matches } |
    ForEach-Object { [double]$_.Groups[1].Value }
```

---

## 2. Playwright Integration (PowerShell 7+)

```powershell
# Install Playwright for PowerShell
Install-Module Microsoft.Playwright -Force

# Headless browser automation
function Invoke-PlaywrightScrape {
    param(
        [string]$Url,
        [string]$Selector,
        [scriptblock]$Action = $null,
        [switch]$Headless
    )
    
    $playwright = [Microsoft.Playwright.Playwright]::CreateAsync().GetAwaiter().GetResult()
    $browser    = $playwright.Chromium.LaunchAsync(@{
        Headless = $Headless.IsPresent -or $true
    }).GetAwaiter().GetResult()
    
    try {
        $page = $browser.NewPageAsync().GetAwaiter().GetResult()
        $page.GotoAsync($Url).GetAwaiter().GetResult()
        
        # Wait for selector
        $page.WaitForSelectorAsync($Selector).GetAwaiter().GetResult()
        
        if ($Action) { & $Action $page }
        
        # Extract data
        $elements = $page.QuerySelectorAllAsync($Selector).GetAwaiter().GetResult()
        $data = foreach ($el in $elements) {
            [PSCustomObject]@{
                Text      = $el.InnerTextAsync().GetAwaiter().GetResult()
                HTML      = $el.InnerHTMLAsync().GetAwaiter().GetResult()
                Attribute = $el.GetAttributeAsync('href').GetAwaiter().GetResult()
            }
        }
        return $data
        
    } finally {
        $browser.CloseAsync().GetAwaiter().GetResult()
        $playwright.Dispose()
    }
}

# Scrape dynamically loaded content
$articles = Invoke-PlaywrightScrape `
    -Url      'https://news.example.com' `
    -Selector 'article.news-item' `
    -Headless `
    -Action   {
        param($page)
        # Scroll to load more content
        $page.EvaluateAsync('window.scrollTo(0, document.body.scrollHeight)').GetAwaiter().GetResult()
        Start-Sleep -Milliseconds 2000
    }
```

---

## 3. Rate-Limited Scraper

```powershell
class RateLimitedScraper {
    [int]$RequestsPerSecond
    [hashtable]$Cache = @{}
    hidden [datetime]$_lastRequest = [datetime]::MinValue
    hidden [System.Collections.Generic.Queue[datetime]]$_requestTimes
    
    RateLimitedScraper([int]$rps = 2) {
        $this.RequestsPerSecond = $rps
        $this._requestTimes     = [System.Collections.Generic.Queue[datetime]]::new()
    }
    
    hidden [void] WaitForRateLimit() {
        $now      = [datetime]::Now
        $window   = 1.0 / $this.RequestsPerSecond
        $elapsed  = ($now - $this._lastRequest).TotalSeconds
        if ($elapsed -lt $window) {
            Start-Sleep -Milliseconds (($window - $elapsed) * 1000)
        }
        $this._lastRequest = [datetime]::Now
    }
    
    [object] Fetch([string]$url, [switch]$UseCache) {
        if ($UseCache -and $this.Cache[$url]) {
            return $this.Cache[$url]
        }
        
        $this.WaitForRateLimit()
        
        $result = Invoke-WebRequest -Uri $url -UseBasicParsing
        if ($UseCache) { $this.Cache[$url] = $result }
        return $result
    }
    
    [object[]] FetchAll([string[]]$urls, [switch]$UseCache) {
        return $urls | ForEach-Object { $this.Fetch($_, -UseCache:$UseCache.IsPresent) }
    }
}

$scraper = [RateLimitedScraper]::new(2)  # 2 req/sec

$urls = 1..20 | ForEach-Object { "https://api.example.com/item/$_" }
$results = $scraper.FetchAll($urls)

$data = $results | ForEach-Object {
    $_.Content | ConvertFrom-Json
} | Where-Object { $_.active }

Write-Host "Scraped $($data.Count) active items"
```

---

## 4. Data Export Pipeline

```powershell
# Export scraped data in multiple formats
function Export-ScrapedData {
    param(
        [object[]]$Data,
        [string]$BaseName = 'scraped-data',
        [string[]]$Formats = @('CSV','JSON','Excel')
    )
    
    $timestamp = Get-Date -Format 'yyyyMMdd-HHmmss'
    $exports   = @{}
    
    if ('CSV' -in $Formats) {
        $path = "$BaseName-$timestamp.csv"
        $Data | Export-Csv $path -NoTypeInformation -Encoding UTF8
        $exports['CSV'] = $path
        Write-Host "CSV: $path" -ForegroundColor Green
    }
    
    if ('JSON' -in $Formats) {
        $path = "$BaseName-$timestamp.json"
        $Data | ConvertTo-Json -Depth 10 | Set-Content $path -Encoding UTF8
        $exports['JSON'] = $path
        Write-Host "JSON: $path" -ForegroundColor Green
    }
    
    if ('XML' -in $Formats) {
        $path = "$BaseName-$timestamp.xml"
        $Data | Export-Clixml $path
        $exports['XML'] = $path
        Write-Host "XML: $path" -ForegroundColor Green
    }
    
    return $exports
}

$scrapedItems = @(
    @{ id=1; title='Item 1'; price=9.99; category='Electronics'; inStock=$true }
    @{ id=2; title='Item 2'; price=19.99; category='Books';       inStock=$false }
) | ForEach-Object { [PSCustomObject]$_ }

Export-ScrapedData -Data $scrapedItems -BaseName './output/products' -Formats @('CSV','JSON')
```

---

**ก่อนหน้า ← [Part 72](Part-72.md) | ต่อไป → [Part 74: Database Abstraction Layer](Part-74.md)**
