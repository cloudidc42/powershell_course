# Part 53: SecureString และ Secrets Management

> **ระดับ**: 🟡 Professional | **เวลา**: ~3 ชั่วขนึ่ง

---

## 1. SecureString

```powershell
# SecureString = encrypted string ใน memory
$secure = ConvertTo-SecureString 'MyP@ssword' -AsPlainText -Force
$secure | Get-Member

# อ่านกลับ (decrypt)
function ConvertFrom-SecureStringToPlain {
    param([securestring]$SecureString)
    [System.Net.NetworkCredential]::new('', $SecureString).Password
}
$plain = ConvertFrom-SecureStringToPlain $secure
Write-Host $plain

# Encrypt กับ DPAPI (ผูกกับ user account)
$encrypted = $secure | ConvertFrom-SecureString  # encrypted hex string
$encrypted | Set-Content 'C:\Secrets\password.enc'

# เพิ่มกลับ
$enc = Get-Content 'C:\Secrets\password.enc'
$secure2 = $enc | ConvertTo-SecureString

# PSCredential
$cred = [PSCredential]::new('username', $secure)
$cred.GetNetworkCredential().Password

# เก็บ credentials เป็น XML
$cred | Export-Clixml 'C:\Secrets\cred.xml'
$cred2 = Import-Clixml 'C:\Secrets\cred.xml'
```

---

## 2. Microsoft.PowerShell.SecretManagement

```powershell
# ติดตั้ง
Install-Module Microsoft.PowerShell.SecretManagement -Force
Install-Module Microsoft.PowerShell.SecretStore -Force  # local vault

# Setup vault
Register-SecretVault -Name 'LocalStore' -ModuleName Microsoft.PowerShell.SecretStore
Set-SecretStoreConfiguration -Authentication Password -PasswordTimeout (New-TimeSpan -Hours 8)

# Store secrets
Set-Secret -Vault 'LocalStore' -Name 'DbPassword' -Secret 'P@ssw0rd123!'
Set-Secret -Vault 'LocalStore' -Name 'ApiKey' -Secret 'sk-1234567890abcdef'
Set-Secret -Vault 'LocalStore' -Name 'DbCred' -Secret (Get-Credential)

# Retrieve
$dbPass = Get-Secret -Vault 'LocalStore' -Name 'DbPassword' -AsPlainText
$apiKey = Get-Secret -Vault 'LocalStore' -Name 'ApiKey' -AsPlainText

# List
Get-SecretInfo -Vault 'LocalStore' | Format-Table Name, Type

# Remove
Remove-Secret -Vault 'LocalStore' -Name 'OldApiKey'

# Azure Key Vault integration
# Install-Module Az.KeyVault
# $secret = Get-AzKeyVaultSecret -VaultName 'myvault' -Name 'DbPassword' -AsPlainText
```

---

## 3. HashiCorp Vault

```powershell
# HashiCorp Vault REST API
function New-VaultClient {
    param([string]$Address = 'http://vault.example.com:8200', [string]$Token)
    @{ Address = $Address; Token = $Token }
}

function Get-VaultSecret {
    param($Client, [string]$Path)
    $resp = Invoke-RestMethod `
        -Uri "$($Client.Address)/v1/$Path" `
        -Headers @{'X-Vault-Token' = $Client.Token} `
        -Method GET
    return $resp.data
}

function Set-VaultSecret {
    param($Client, [string]$Path, [hashtable]$Data)
    Invoke-RestMethod `
        -Uri "$($Client.Address)/v1/$Path" `
        -Headers @{'X-Vault-Token' = $Client.Token} `
        -Method POST `
        -Body ($Data | ConvertTo-Json) `
        -ContentType 'application/json'
}

# ใช้งาน
$vault  = New-VaultClient -Token $env:VAULT_TOKEN
$dbcred = Get-VaultSecret $vault 'secret/database'
Write-Host "DB User: $($dbcred.username)"
```

---

## 4. Environment-based Secrets

```powershell
# ใช้ environment variables สำหรับ CI/CD
function Get-RequiredSecret {
    param([string[]]$Names)
    $secrets = @{}
    $missing = @()
    
    foreach ($name in $Names) {
        $val = [System.Environment]::GetEnvironmentVariable($name)
        if (!$val) { $missing += $name }
        else { $secrets[$name] = $val }
    }
    
    if ($missing) {
        throw "Missing required environment variables: $($missing -join ', ')"
    }
    
    return $secrets
}

$secrets = Get-RequiredSecret @('DB_PASSWORD','API_KEY','JWT_SECRET')
# ถ้าไม่มีตัวไหนตัวหนึ่ง จะ throw error

# .env parser
function Import-DotEnv {
    param([string]$Path = '.env')
    if (!(Test-Path $Path)) { return }
    Get-Content $Path | ForEach-Object {
        if ($_ -match '^([A-Z_][A-Z0-9_]*)=(.*)$') {
            [System.Environment]::SetEnvironmentVariable($Matches[1], $Matches[2])
        }
    }
}

Import-DotEnv '.env'
Write-Host $env:DB_PASSWORD
```

---

**ก่อนหน้า ← [Part 52](Part-52.md) | ต่อไป → [Part 54: Logging & SIEM](Part-54.md)**
