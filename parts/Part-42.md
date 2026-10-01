# Part 42: AMSI และ Script Security

> **ระดับ**: 🔴 Professional/Security | **เวลย**: ~5 ชั่วขนึ่ง

> ⚠️ **สำหรับ**: Blue Team, การทำความเข้าใจ AMSI เพื่อป้องกันตัวเอง, CTF, Security Research **เท่านั้น**

---

## 1. AMSI คืออะไร?

AMSI = Antimalware Scan Interface— API ของ Windows ที่ส่ง content ไปยัง AV engine ก่อนเรียกใช้

```
PowerShell  -->  amsi.dll  -->  WinDefend (or 3rd party AV)
                               |
                               +--> AMSI_RESULT_NOT_DETECTED   (OK)
                               +--> AMSI_RESULT_DETECTED       (Block!)
```

```powershell
# สาย AMSI scan ของ PS
# PS ใช้ AmsiScanBuffer / AmsiScanString
# เรียกทุกครั้ง script/command ถูก invoke

# เสริม AMSI ใน process ของตัวเอง (สำหรับ AV product)
$amsi = [System.Runtime.InteropServices.RuntimeEnvironment]::GetRuntimeDirectory()
# amsi.dll อยู่ใน C:\Windows\System32\

# Test AMSI manually (PoC สำหรับ Blue Team testing)
Add-Type -TypeDefinition @'
public class AmsiTest {
    [System.Runtime.InteropServices.DllImport("amsi.dll")]
    public static extern int AmsiInitialize(string appName, out System.IntPtr amsiContext);
    [System.Runtime.InteropServices.DllImport("amsi.dll")]
    public static extern int AmsiOpenSession(System.IntPtr amsiContext, out System.IntPtr session);
    [System.Runtime.InteropServices.DllImport("amsi.dll", CharSet=System.Runtime.InteropServices.CharSet.Unicode)]
    public static extern int AmsiScanString(System.IntPtr amsiContext, string str, string contentName, System.IntPtr session, out int result);
    [System.Runtime.InteropServices.DllImport("amsi.dll")]
    public static extern void AmsiCloseSession(System.IntPtr amsiContext, System.IntPtr session);
    [System.Runtime.InteropServices.DllImport("amsi.dll")]
    public static extern void AmsiUninitialize(System.IntPtr amsiContext);
}
'@

function Test-AmsiScan {
    param([string]$Content)
    $ctx = [IntPtr]::Zero
    $session = [IntPtr]::Zero
    [AmsiTest]::AmsiInitialize('PSTest', [ref]$ctx) | Out-Null
    [AmsiTest]::AmsiOpenSession($ctx, [ref]$session) | Out-Null
    $result = 0
    [AmsiTest]::AmsiScanString($ctx, $Content, 'test', $session, [ref]$result) | Out-Null
    [AmsiTest]::AmsiCloseSession($ctx, $session)
    [AmsiTest]::AmsiUninitialize($ctx)
    return $result  # 32768 = clean, 32769 = detected
}

Test-AmsiScan 'Write-Host Hello'  # should return 32768 (clean)
```

---

## 2. PowerShell Security Features

```powershell
# Execution Policy
Get-ExecutionPolicy -List
Set-ExecutionPolicy Restricted      # most restrictive
Set-ExecutionPolicy RemoteSigned    # scripts must be signed if from internet
Set-ExecutionPolicy Bypass          # no check (dangerous!)
Set-ExecutionPolicy Undefined       # inherited

# Script Block Logging (Event ID 4104)
# กำหนดใน Group Policy:
# Computer Config > Admin Templates > Windows > PowerShell > Script Block Logging
# หรือผ่าน registry:
New-Item 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging' -Force
Set-ItemProperty 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging' `
    -Name EnableScriptBlockLogging -Value 1

# Module Logging (Event ID 4103)
New-Item 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging' -Force
Set-ItemProperty 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging' `
    -Name EnableModuleLogging -Value 1

# Transcription
New-Item 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\Transcription' -Force
Set-ItemProperty 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\Transcription' `
    -Name EnableTranscripting -Value 1
Set-ItemProperty 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\Transcription' `
    -Name OutputDirectory -Value 'C:\PSLogs'

# Constrained Language Mode (CLM)
$ExecutionContext.SessionState.LanguageMode  # view current mode
# FullLanguage / ConstrainedLanguage / RestrictedLanguage / NoLanguage

# CLM restrictions:
# - No COM objects, .NET types from certain assemblies
# - Limited Add-Type, Invoke-Expression
# - No PInvoke
```

---

## 3. Code Signing

```powershell
# สร้าง code signing certificate
$cert = New-SelfSignedCertificate `
    -Type CodeSigningCert `
    -Subject 'CN=My Code Signing' `
    -CertStoreLocation 'Cert:\CurrentUser\My' `
    -NotAfter (Get-Date).AddYears(3)

# Sign script
Set-AuthenticodeSignature `
    -FilePath 'script.ps1' `
    -Certificate $cert `
    -HashAlgorithm SHA256 `
    -TimestampServer 'http://timestamp.digicert.com'

# Verify signature
$sig = Get-AuthenticodeSignature 'script.ps1'
$sig.Status          # Valid / Invalid / NotSigned
$sig.SignerCertificate.Subject

# Catalog signing (.cat)
New-FileCatalog -CatalogFilePath 'MyModule.cat' -Path '.\MyModule' -CatalogVersion 2
Set-AuthenticodeSignature -FilePath 'MyModule.cat' -Certificate $cert
Test-FileCatalog -CatalogFilePath 'MyModule.cat' -Path '.\MyModule'
```

---

## 4. Just Enough Administration (JEA)

```powershell
# JEA: ให้สิทธิ์เฉพาะ cmdlets ที่จำเป็น

# Role Capability file (WebAdmins.psrc)
New-PSRoleCapabilityFile -Path 'C:\JEA\WebAdmins.psrc'

# แก้ไข .psrc:
# VisibleCmdlets = @(
#     'Restart-Service',
#     @{Name='Set-Service'; Parameters=@{Name='Name'; ValidateSet='W3SVC','WAS'}}
# )
# VisibleFunctions = @()
# VisibleExternalCommands = @('C:\Windows\System32\iisreset.exe')

# Session Config (WebAdminsSession.pssc)
New-PSSessionConfigurationFile `
    -Path 'C:\JEA\WebAdminsSession.pssc' `
    -RunAsVirtualAccount `
    -RoleDefinitions @{
        'DOMAIN\WebAdmins' = @{ RoleCapabilities = 'WebAdmins' }
    } `
    -SessionType RestrictedRemoteServer

# Register session
Register-PSSessionConfiguration `
    -Name 'WebAdminsSession' `
    -Path 'C:\JEA\WebAdminsSession.pssc' `
    -Force

# เชื่อมต่อ
Enter-PSSession -ComputerName server01 -ConfigurationName WebAdminsSession
```

---

**ก่อนหน้า ← [Part 41](Part-41.md) | ต่อไป → [Part 43: AV & EDR Technology](Part-43.md)**
