# Part 61: Terraform & Infrastructure as Code

> **ระดับ**: 🔴 Advanced | **เวลา**: ~5 ชั่วโมง

---

## 1. Terraform Wrapper Class

```powershell
class TerraformRunner {
    [string]$WorkDir
    [hashtable]$Env = @{}
    
    TerraformRunner([string]$workDir) { $this.WorkDir = $workDir }
    
    hidden [string] Run([string[]]$tfArgs) {
        $psi = [System.Diagnostics.ProcessStartInfo]::new('terraform')
        $psi.Arguments              = $tfArgs -join ' '
        $psi.WorkingDirectory       = $this.WorkDir
        $psi.RedirectStandardOutput = $true
        $psi.RedirectStandardError  = $true
        $psi.UseShellExecute        = $false
        foreach ($kv in $this.Env.GetEnumerator()) {
            $psi.EnvironmentVariables[$kv.Key] = $kv.Value
        }
        $proc = [System.Diagnostics.Process]::Start($psi)
        $out  = $proc.StandardOutput.ReadToEnd()
        $err  = $proc.StandardError.ReadToEnd()
        $proc.WaitForExit()
        if ($proc.ExitCode -ne 0) { throw "Terraform failed (exit $($proc.ExitCode)):`n$err" }
        return $out
    }
    
    [void] Init([switch]$Upgrade) {
        $a = @('init','-input=false')
        if ($Upgrade) { $a += '-upgrade' }
        $this.Run($a)
        Write-Host 'Terraform init complete' -ForegroundColor Green
    }
    
    [string] Plan([string]$PlanFile = 'tfplan', [hashtable]$Vars = @{}) {
        $a = @('plan','-input=false',"-out=$PlanFile")
        foreach ($kv in $Vars.GetEnumerator()) { $a += "-var `"$($kv.Key)=$($kv.Value)`"" }
        return $this.Run($a)
    }
    
    [void] Apply([string]$PlanFile = 'tfplan', [switch]$AutoApprove) {
        $a = @('apply','-input=false')
        if ($AutoApprove) { $a += '-auto-approve' } else { $a += $PlanFile }
        $this.Run($a)
        Write-Host 'Apply complete' -ForegroundColor Green
    }
    
    [void] Destroy([switch]$AutoApprove) {
        $a = @('destroy','-input=false')
        if ($AutoApprove) { $a += '-auto-approve' }
        $this.Run($a)
    }
    
    [hashtable] Output() {
        return ($this.Run(@('output','-json')) | ConvertFrom-Json -AsHashtable)
    }
    
    [object] ShowPlan([string]$PlanFile = 'tfplan') {
        return ($this.Run(@('show','-json',$PlanFile)) | ConvertFrom-Json)
    }
}

$tf = [TerraformRunner]::new('./infra/azure')
$tf.Env['ARM_SUBSCRIPTION_ID'] = $env:AZURE_SUBSCRIPTION_ID
$tf.Env['ARM_CLIENT_ID']        = $env:AZURE_CLIENT_ID
$tf.Env['ARM_CLIENT_SECRET']    = $env:AZURE_CLIENT_SECRET
$tf.Env['ARM_TENANT_ID']        = $env:AZURE_TENANT_ID

$tf.Init(-Upgrade)
$tf.Plan('tfplan', @{ environment='production'; location='eastus' })

$plan    = $tf.ShowPlan('tfplan')
$changes = $plan.resource_changes | Where-Object { $_.change.actions -ne @('no-op') }
Write-Host "Resources to change: $($changes.Count)" -ForegroundColor Yellow
$changes | Format-Table address, @{n='Action';e={$_.change.actions -join '+'}}

$tf.Apply('tfplan', -AutoApprove)
$out = $tf.Output()
Write-Host "App URL: $($out.app_url.value)"
```

---

## 2. Terraform Variables Generator

```powershell
function New-TerraformVars {
    param([hashtable]$Variables, [string]$OutputFile)
    
    $lines = $Variables.GetEnumerator() | ForEach-Object {
        $val = switch ($_.Value.GetType().Name) {
            'String'  { "`"$($_.Value)`"" }
            'Boolean' { $_.Value.ToString().ToLower() }
            default   { $_.Value }
        }
        "$($_.Key) = $val"
    }
    $lines | Set-Content $OutputFile
    Write-Host "Wrote $OutputFile" -ForegroundColor Green
}

New-TerraformVars -OutputFile './infra/prod.tfvars' -Variables @{
    environment       = 'production'
    location          = 'eastus'
    vm_count          = 3
    enable_monitoring = $true
}
```

---

## 3. CI/CD Pipeline

```powershell
function Invoke-TerraformPipeline {
    param(
        [string]$WorkDir,
        [string]$Environment,
        [hashtable]$Vars       = @{},
        [switch]$AutoApprove,
        [string]$NotifyWebhook
    )
    
    $tf     = [TerraformRunner]::new($WorkDir)
    $status = 'started'
    
    try {
        Write-Host "`n=== Terraform Pipeline: $Environment ===" -ForegroundColor Cyan
        
        $tf.Init()
        
        $planFile = "plan-$Environment-$(Get-Date -Format yyyyMMddHHmmss)"
        $tf.Plan($planFile, $Vars + @{ environment=$Environment })
        
        $plan    = $tf.ShowPlan($planFile)
        $adds    = @($plan.resource_changes | Where-Object { 'create' -in $_.change.actions }).Count
        $updates = @($plan.resource_changes | Where-Object { 'update' -in $_.change.actions }).Count
        $deletes = @($plan.resource_changes | Where-Object { 'delete' -in $_.change.actions }).Count
        Write-Host "Plan: +$adds adds, ~$updates updates, -$deletes destroys" -ForegroundColor Cyan
        
        if (-not $AutoApprove -and ($deletes -gt 0)) {
            $confirm = Read-Host "Destructive changes! Type 'yes' to continue"
            if ($confirm -ne 'yes') { throw 'Aborted by user' }
        }
        
        $tf.Apply($planFile, -AutoApprove:$AutoApprove)
        $status = 'success'
        Write-Host 'Pipeline complete!' -ForegroundColor Green
        return $tf.Output()
        
    } catch {
        $status = "failed: $_"
        Write-Error $_
    } finally {
        if ($NotifyWebhook) {
            $msg = @{ environment=$Environment; status=$status; ts=[datetime]::UtcNow } | ConvertTo-Json
            Invoke-RestMethod -Uri $NotifyWebhook -Method POST -Body $msg -ContentType 'application/json'
        }
    }
}

Invoke-TerraformPipeline -WorkDir './infra/azure' -Environment 'production' `
    -Vars @{ vm_size='Standard_D2s_v3'; replicas=3 } -AutoApprove
```

---

**ก่อนหน้า ← [Part 60](Part-60.md) | ต่อไป → [Part 62: Monitoring & Alerting](Part-62.md)**
