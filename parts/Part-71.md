# Part 71: Data Processing & Analytics

> **ระดับ**: 🔴 Advanced | **เวลา**: ~4 ชั่วโมง

---

## 1. Statistical Analysis

```powershell
function Get-Statistics {
    param([double[]]$Data)
    
    $sorted = $Data | Sort-Object
    $n      = $Data.Count
    $sum    = ($Data | Measure-Object -Sum).Sum
    $mean   = $sum / $n
    
    # Variance & StdDev
    $variance = ($Data | ForEach-Object { [math]::Pow($_ - $mean, 2) } | Measure-Object -Sum).Sum / ($n - 1)
    $stdDev   = [math]::Sqrt($variance)
    
    # Median
    $median = if ($n % 2 -eq 0) {
        ($sorted[$n/2-1] + $sorted[$n/2]) / 2
    } else {
        $sorted[[int]($n/2)]
    }
    
    # Percentiles
    function Get-Percentile([double[]]$s, [double]$p) {
        $index = [int][math]::Ceiling($p/100 * $s.Count) - 1
        return $s[[Math]::Max(0, [Math]::Min($index, $s.Count-1))]
    }
    
    [PSCustomObject]@{
        Count  = $n
        Sum    = [math]::Round($sum, 4)
        Min    = $sorted[0]
        Max    = $sorted[-1]
        Mean   = [math]::Round($mean, 4)
        Median = [math]::Round($median, 4)
        StdDev = [math]::Round($stdDev, 4)
        P25    = Get-Percentile $sorted 25
        P75    = Get-Percentile $sorted 75
        P95    = Get-Percentile $sorted 95
        P99    = Get-Percentile $sorted 99
    }
}

# Time series analysis
function Get-MovingAverage {
    param([double[]]$Data, [int]$Window)
    $result = @()
    for ($i = $Window-1; $i -lt $Data.Count; $i++) {
        $window = $Data[($i-$Window+1)..$i]
        $result += [math]::Round(($window | Measure-Object -Average).Average, 4)
    }
    return $result
}

# Sample usage
$data = 1..1000 | ForEach-Object { [double](Get-Random -Minimum 1 -Maximum 100) }
$stats = Get-Statistics -Data $data
$stats | Format-List

$sma7  = Get-MovingAverage $data 7
$sma30 = Get-MovingAverage $data 30
```

---

## 2. Data Pivot & Aggregation

```powershell
# Pivot table
function New-PivotTable {
    param(
        [object[]]$Data,
        [string]$RowField,
        [string]$ColField,
        [string]$ValueField,
        [string]$AggFunc = 'Sum'  # Sum, Count, Average, Max, Min
    )
    
    $rows   = $Data | Select-Object -ExpandProperty $RowField  -Unique | Sort-Object
    $cols   = $Data | Select-Object -ExpandProperty $ColField  -Unique | Sort-Object
    $pivot  = [ordered]@{}
    
    foreach ($row in $rows) {
        $pivot[$row] = [ordered]@{ $RowField = $row }
        
        foreach ($col in $cols) {
            $values = $Data | Where-Object { $_.$RowField -eq $row -and $_.$ColField -eq $col } |
                ForEach-Object { [double]$_.$ValueField }
            
            $pivot[$row][$col] = if ($values) {
                switch ($AggFunc) {
                    'Sum'     { [math]::Round(($values | Measure-Object -Sum).Sum, 2) }
                    'Count'   { $values.Count }
                    'Average' { [math]::Round(($values | Measure-Object -Average).Average, 2) }
                    'Max'     { ($values | Measure-Object -Maximum).Maximum }
                    'Min'     { ($values | Measure-Object -Minimum).Minimum }
                }
            } else { 0 }
        }
        
        # Row total
        $pivot[$row]['Total'] = [math]::Round(($pivot[$row].Values | Where-Object { $_ -is [double] } | Measure-Object -Sum).Sum, 2)
    }
    
    return $pivot.Values | ForEach-Object { [PSCustomObject]$_ }
}

# Sample: Sales pivot
$sales = @(
    @{Region='North'; Product='Widget'; Amount=1000}
    @{Region='North'; Product='Gadget'; Amount=500}
    @{Region='South'; Product='Widget'; Amount=800}
    @{Region='South'; Product='Gadget'; Amount=1200}
    @{Region='East';  Product='Widget'; Amount=600}
) | ForEach-Object { [PSCustomObject]$_ }

$pivot = New-PivotTable -Data $sales -RowField 'Region' -ColField 'Product' -ValueField 'Amount'
$pivot | Format-Table
```

---

## 3. JSON และ Data Transformation

```powershell
# Flatten nested JSON
function ConvertTo-FlatObject {
    param([object]$InputObject, [string]$Prefix = '', [string]$Separator = '.')
    
    $result = [ordered]@{}
    
    foreach ($prop in $InputObject.PSObject.Properties) {
        $key = if ($Prefix) { "$Prefix$Separator$($prop.Name)" } else { $prop.Name }
        
        if ($prop.Value -is [PSCustomObject] -or $prop.Value -is [hashtable]) {
            $nested = ConvertTo-FlatObject -InputObject $prop.Value -Prefix $key -Separator $Separator
            foreach ($kv in $nested.GetEnumerator()) { $result[$kv.Key] = $kv.Value }
        } elseif ($prop.Value -is [array]) {
            for ($i = 0; $i -lt $prop.Value.Count; $i++) {
                $arrKey = "$key[$i]"
                if ($prop.Value[$i] -is [PSCustomObject]) {
                    $nested = ConvertTo-FlatObject -InputObject $prop.Value[$i] -Prefix $arrKey -Separator $Separator
                    foreach ($kv in $nested.GetEnumerator()) { $result[$kv.Key] = $kv.Value }
                } else {
                    $result[$arrKey] = $prop.Value[$i]
                }
            }
        } else {
            $result[$key] = $prop.Value
        }
    }
    return [PSCustomObject]$result
}

$nested = @'
{
    "user": { "id": 1, "name": "Alice", "address": { "city": "Bangkok", "zip": "10100" } },
    "orders": [{ "id": 101, "amount": 99.99 }, { "id": 102, "amount": 199.99 }]
}
'@ | ConvertFrom-Json

$flat = ConvertTo-FlatObject $nested
$flat | Format-List

# JMESPath-style query
function Select-JsonPath {
    param([object]$Data, [string]$Path)
    $parts = $Path -split '\.'
    $current = $Data
    foreach ($part in $parts) {
        if ($part -match '^(.+)\[(\d+)\]$') {
            $current = $current.($Matches[1])[$Matches[2]]
        } else {
            $current = $current.$part
        }
    }
    return $current
}

Select-JsonPath $nested 'user.address.city'  # Bangkok
Select-JsonPath $nested 'orders[0].amount'   # 99.99
```

---

**ก่อนหน้า ← [Part 70](Part-70.md) | ต่อไป → [Part 72: GUI Applications](Part-72.md)**
