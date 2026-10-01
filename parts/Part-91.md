# Part 91: Real-World Case Study — Cloud Migration

> **ระดับ**: 🟠 World-Class | **เวลา**: ~6 ชั่วข็น (+workshop)

---

## Scenario: On-Prem → Azure Migration Automation

**ต้องการ**:
1. Discovery: สำรวจ VM/Server ออนเปรมทั้งหมด
2. Assessment: ประเมิน compatibility, cost, risk
3. Migration: ย้ายอัตโนมัติ
4. Validation: ตรวจสอบหลังย้าย

---

## 1. Discovery และ Assessment

```powershell
class MigrationAssessor {
    [System.Collections.Generic.List[PSCustomObject]]$Servers
    
    MigrationAssessor() {
        $this.Servers = [System.Collections.Generic.List[PSCustomObject]]::new()
    }
    
    [void] DiscoverOnPrem([string[]]$ComputerNames) {
        $cred = Get-Credential
        
        $jobs = $ComputerNames | ForEach-Object {
            $computer = $_
            Start-Job {
                param($Computer, $Cred)
                try {
                    $session = New-PSSession -ComputerName $Computer -Credential $Cred -ErrorAction Stop
                    
                    $info = Invoke-Command -Session $session {
                        $os  = Get-CimInstance Win32_OperatingSystem
                        $cpu = Get-CimInstance Win32_Processor | Select-Object -First 1
                        $mem = [math]::Round($os.TotalVisibleMemorySize / 1MB, 0)
                        $disks = Get-Volume | Where-Object { $_.DriveLetter } |
                            Select-Object DriveLetter,
                                @{n='SizeGB';e={[math]::Round($_.Size/1GB,0)}},
                                @{n='FreeGB';e={[math]::Round($_.SizeRemaining/1GB,0)}}
                        
                        @{
                            OS       = $os.Caption
                            Build    = $os.BuildNumber
                            RAMGB    = $mem
                            CPUCores = $cpu.NumberOfCores
                            CPUGHZ   = [math]::Round($cpu.MaxClockSpeed/1000, 1)
                            Disks    = $disks
                            TotalDiskGB = ($disks | Measure-Object -Property SizeGB -Sum).Sum
                            Services = @(Get-Service | Where-Object { $_.StartType -eq 'Automatic' -and $_.Status -eq 'Running' }).Count
                            Uptime   = (Get-Date) - $os.LastBootUpTime
                        }
                    }
                    
                    Remove-PSSession $session
                    return @{ Computer=$Computer; Info=$info; Error=$null }
                } catch {
                    return @{ Computer=$Computer; Info=$null; Error=$_.Exception.Message }
                }
            } -ArgumentList $computer, $cred
        }
        
        $jobs | Wait-Job | ForEach-Object {
            $result = Receive-Job $_
            if ($result.Info) {
                $this.Servers.Add([PSCustomObject]@{
                    Name       = $result.Computer
                    OS         = $result.Info.OS
                    Build      = $result.Info.Build
                    RAMGB      = $result.Info.RAMGB
                    CPUCores   = $result.Info.CPUCores
                    TotalDiskGB= $result.Info.TotalDiskGB
                    Services   = $result.Info.Services
                    UptimeDays = [math]::Round($result.Info.Uptime.TotalDays, 0)
                    Error      = $null
                })
            } else {
                $this.Servers.Add([PSCustomObject]@{
                    Name  = $result.Computer
                    Error = $result.Error
                })
            }
            Remove-Job $_
        }
    }
    
    [PSCustomObject[]] AssessForAzure() {
        return $this.Servers | Where-Object { -not $_.Error } | ForEach-Object {
            $server = $_
            
            # Recommend Azure VM SKU
            $sku = if     ($server.CPUCores -le 2 -and $server.RAMGB -le 8)   { 'Standard_B2s' }
                   elseif ($server.CPUCores -le 4 -and $server.RAMGB -le 16)  { 'Standard_D2s_v5' }
                   elseif ($server.CPUCores -le 8 -and $server.RAMGB -le 32)  { 'Standard_D4s_v5' }
                   else                                                         { 'Standard_D8s_v5' }
            
            # Estimate monthly cost (rough)
            $skuCost = @{
                'Standard_B2s'   = 35
                'Standard_D2s_v5'= 70
                'Standard_D4s_v5'= 140
                'Standard_D8s_v5'= 280
            }
            
            # Migration risk
            $risk = if ($server.Build -lt 9200)  { 'HIGH - EOL OS' }     # < Server 2012
                    elseif ($server.Build -lt 14393) { 'MEDIUM - Older OS' } # < Server 2016
                    else                              { 'LOW' }
            
            [PSCustomObject]@{
                Server          = $server.Name
                RecommendedSKU  = $sku
                EstimatedMonthlyCost = $skuCost[$sku]
                DiskNeededGB    = [math]::Ceiling($server.TotalDiskGB * 1.2)  # 20% buffer
                Risk            = $risk
                ReadyToMigrate  = $risk -eq 'LOW'
                Notes           = if ($server.UptimeDays -gt 180) { 'Review pending Windows Updates' } else { '' }
            }
        }
    }
    
    [void] ExportReport([string]$Path) {
        $assessment = $this.AssessForAzure()
        $totalCost  = ($assessment | Measure-Object -Property EstimatedMonthlyCost -Sum).Sum
        $highRisk   = @($assessment | Where-Object { $_.Risk -like 'HIGH*' }).Count
        
        $report = @{
            GeneratedAt         = [datetime]::UtcNow.ToString('o')
            TotalServers        = $assessment.Count
            ReadyToMigrate      = @($assessment | Where-Object ReadyToMigrate).Count
            HighRiskServers     = $highRisk
            TotalMonthlyCostUSD = $totalCost
            Servers             = $assessment
        }
        
        $report | ConvertTo-Json -Depth 5 | Set-Content $Path
        Write-Host "Assessment saved to $Path" -ForegroundColor Green
        Write-Host "  Total servers: $($report.TotalServers)"
        Write-Host "  Ready to migrate: $($report.ReadyToMigrate)"
        Write-Host "  High risk: $highRisk" -ForegroundColor $(if ($highRisk -gt 0) {'Red'} else {'Green'})
        Write-Host "  Est. monthly cost: $$totalCost/month"
    }
}

# Usage
$assessor = [MigrationAssessor]::new()
$assessor.DiscoverOnPrem(@('SRV-APP-01','SRV-APP-02','SRV-DB-01','SRV-WEB-01','SRV-FILE-01'))
$assessor.ExportReport('./migration-assessment.json')
```

---

## 2. Automated Migration Execution

```powershell
function Invoke-AzureMigration {
    param(
        [PSCustomObject]$ServerAssessment,
        [string]$SubscriptionId,
        [string]$ResourceGroup,
        [string]$Location = 'eastus',
        [switch]$WhatIf
    )
    
    $server = $ServerAssessment
    if (-not $server.ReadyToMigrate) {
        Write-Warning "$($server.Server) is not ready: $($server.Risk)"
        return
    }
    
    Write-Host "Migrating: $($server.Server)" -ForegroundColor Cyan
    
    if ($WhatIf) {
        Write-Host "  [WhatIf] Would create VM: $($server.RecommendedSKU) in $ResourceGroup" -ForegroundColor Yellow
        return
    }
    
    # 1. Create disk from on-prem VHD upload (assumes already uploaded to blob)
    $diskConfig = New-AzDiskConfig `
        -Location      $Location `
        -DiskSizeGB    $server.DiskNeededGB `
        -SkuName       'Premium_LRS' `
        -CreateOption  'Import' `
        -StorageAccountId (Get-AzStorageAccount -ResourceGroupName $ResourceGroup | Select-Object -First 1).Id
    
    $disk = New-AzDisk -DiskName "$($server.Server)-osdisk" -Disk $diskConfig -ResourceGroupName $ResourceGroup
    
    # 2. Create VM from disk
    $vmConfig = New-AzVMConfig -VMName $server.Server -VMSize $server.RecommendedSKU
    $vmConfig = Set-AzVMOSDisk   -VM $vmConfig -ManagedDiskId $disk.Id -CreateOption Attach -Windows
    $vmConfig = Add-AzVMNetworkInterface -VM $vmConfig -Id (Get-AzNetworkInterface -ResourceGroupName $ResourceGroup | Select-Object -First 1).Id
    
    New-AzVM -VM $vmConfig -ResourceGroupName $ResourceGroup -Location $Location
    
    Write-Host "  VM created: $($server.Server)" -ForegroundColor Green
    
    # 3. Enable monitoring
    Set-AzVMExtension `
        -ResourceGroupName $ResourceGroup `
        -VMName            $server.Server `
        -Name              'MicrosoftMonitoringAgent' `
        -Publisher         'Microsoft.EnterpriseCloud.Monitoring' `
        -ExtensionType     'MicrosoftMonitoringAgent' `
        -TypeHandlerVersion '1.0' `
        -Settings          @{ workspaceId=$env:LAW_ID } `
        -ProtectedSettings @{ workspaceKey=$env:LAW_KEY }
    
    Write-Host "  Monitoring enabled" -ForegroundColor Green
}

# Migrate all ready servers
$assessment = Get-Content './migration-assessment.json' | ConvertFrom-Json
$readyServers = $assessment.Servers | Where-Object { $_.ReadyToMigrate }

foreach ($server in $readyServers) {
    Invoke-AzureMigration `
        -ServerAssessment $server `
        -SubscriptionId   $env:AZURE_SUBSCRIPTION `
        -ResourceGroup    'migration-rg' `
        -WhatIf
}
```

---

**ก่อนหน้า ← [Part 90](Part-90.md) | ต่อไป → [Part 92: Service Mesh และ Observability](Part-92.md)**
