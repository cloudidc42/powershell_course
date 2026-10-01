# Part 17: .NET Integration

> **ระดับ**: 🟡 Intermediate | **เวลา**: ~4 ชั่วโมง

---

## 1. .NET Types และ Static Methods

```powershell
# เรียก static method
[System.Math]::PI
[System.Math]::Round(3.14159, 2)    # 3.14
[System.Math]::Abs(-42)             # 42
[System.Math]::Pow(2, 10)           # 1024
[System.Math]::Sqrt(16)             # 4
[System.Math]::Log(100, 10)         # 2 (log base 10)
[System.Math]::Max(5, 10)           # 10
[System.Math]::Clamp(15, 0, 10)     # 10 (PS7+)

# String methods
[System.String]::Format("{0} {1}", 'Hello', 'World')
[System.String]::IsNullOrEmpty('')  # True
[System.String]::Join(', ', @('a','b','c'))  # a, b, c

# Environment
[System.Environment]::MachineName
[System.Environment]::UserName
[System.Environment]::OSVersion
[System.Environment]::ProcessorCount
[System.Environment]::GetEnvironmentVariable('PATH')
[System.Environment]::Is64BitOperatingSystem
[System.Environment]::Is64BitProcess

# Convert
[System.Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes('Hello'))
[System.Convert]::ToString(255, 16)  # ff
[System.Convert]::ToInt32('FF', 16)  # 255
```

---

## 2. Reflection

```powershell
# โหลด assembly
$asm = [System.Reflection.Assembly]::GetAssembly([System.String])
$asm.FullName
$asm.Location

# Explore type
$type = [System.Collections.Generic.List[int]]
$type.GetMethods() | Select-Object Name | Sort-Object Name -Unique
$type.GetProperties() | Select-Object Name
$type.GetConstructors() | ForEach-Object { $_.GetParameters() | ForEach-Object { $_.ParameterType } }

# อ่าน/เขียน private member
$list = [System.Collections.Generic.List[int]]::new()
$list.Add(1); $list.Add(2); $list.Add(3)

# อ่าน private field
$field = $list.GetType().GetField('_items', [System.Reflection.BindingFlags]'NonPublic,Instance')
$items = $field.GetValue($list)
# items = internal array

# Dynamic method call
$method = $list.GetType().GetMethod('BinarySearch', @([int]))
$list.Sort()
$index = $method.Invoke($list, @([int]2))  # 1

# Create instance dynamically
$type = [System.Text.StringBuilder]
$obj  = [System.Activator]::CreateInstance($type, 100)  # capacity=100
$obj.Append('Hello') | Out-Null
$obj.ToString()  # Hello
```

---

## 3. Delegates และ Events

```powershell
# สร้าง delegate
$action = [System.Action[string]]{ param($s) Write-Host "Action: $s" }
$action.Invoke('Hello')   # Action: Hello

$func = [System.Func[int,int,int]]{ param($a,$b) $a + $b }
$func.Invoke(3, 4)   # 7

$pred = [System.Predicate[int]]{ param($n) $n % 2 -eq 0 }
$pred.Invoke(4)   # True

# Register/Unregister events
$timer = [System.Timers.Timer]::new(1000)  # 1 second

$job = Register-ObjectEvent -InputObject $timer -EventName Elapsed -Action {
    Write-Host "Tick! $(Get-Date -Format 'HH:mm:ss')"
}

$timer.Start()
Start-Sleep 5
$timer.Stop()
Unregister-Event -SourceIdentifier $job.Name
$timer.Dispose()

# เอา event เข้า variable
$result = $null
$timer2 = [System.Timers.Timer]::new(500)
Register-ObjectEvent $timer2 -EventName Elapsed -SourceIdentifier 'myTimer' -Action {
    $script:result = "Timer fired at $(Get-Date)"
} | Out-Null

$timer2.AutoReset = $false
$timer2.Start()
Wait-Event -SourceIdentifier 'myTimer' -Timeout 2
Unregister-Event 'myTimer'
Write-Host $result
```

---

## 4. P/Invoke (Native API)

```powershell
# เรียก Win32 API ผ่าน P/Invoke
$code = @'
using System;
using System.Runtime.InteropServices;

public class NativeMethods {
    [DllImport("kernel32.dll")]
    public static extern bool Beep(int frequency, int duration);
    
    [DllImport("user32.dll")]
    public static extern IntPtr GetForegroundWindow();
    
    [DllImport("user32.dll")]
    public static extern bool ShowWindow(IntPtr hWnd, int nCmdShow);
    
    [DllImport("kernel32.dll")]
    public static extern int GetCurrentProcessId();
}
'@

Add-Type -TypeDefinition $code

[NativeMethods]::Beep(1000, 500)   # 1kHz tone for 500ms
[NativeMethods]::GetCurrentProcessId()

# เรียก via P/Invoke ด้วย Add-Type
Add-Type @'
    using System;
    using System.Runtime.InteropServices;
    
    public class Meminfo {
        [StructLayout(LayoutKind.Sequential)]
        public struct MEMORYSTATUSEX {
            public uint dwLength;
            public uint dwMemoryLoad;
            public ulong ullTotalPhys;
            public ulong ullAvailPhys;
            // ... more fields
        }
        
        [DllImport("kernel32.dll")]
        public static extern bool GlobalMemoryStatusEx(ref MEMORYSTATUSEX lpBuffer);
    }
'@

$mem = New-Object Meminfo+MEMORYSTATUSEX
$mem.dwLength = [System.Runtime.InteropServices.Marshal]::SizeOf($mem)
[Meminfo]::GlobalMemoryStatusEx([ref]$mem) | Out-Null

Write-Host "Total RAM: $([math]::Round($mem.ullTotalPhys/1GB,2)) GB"
Write-Host "Available: $([math]::Round($mem.ullAvailPhys/1GB,2)) GB"
Write-Host "Usage: $($mem.dwMemoryLoad)%"
```

---

## 5. Compile C# ใน PowerShell

```powershell
# Add-Type ใช้คอมไพล์ C#
$csharp = @'
using System;
using System.Collections.Generic;
using System.Linq;

namespace MyTools {
    public class StringUtils {
        public static string TitleCase(string input) {
            if (string.IsNullOrEmpty(input)) return input;
            var words = input.ToLower().Split(' ');
            return string.Join(" ", words.Select(w => 
                w.Length > 0 ? char.ToUpper(w[0]) + w.Substring(1) : w));
        }
        
        public static IEnumerable<string> Chunk(string text, int size) {
            for (int i = 0; i < text.Length; i += size)
                yield return text.Substring(i, Math.Min(size, text.Length - i));
        }
        
        public static bool IsPalindrome(string s) {
            var clean = new string(s.ToLower().Where(char.IsLetterOrDigit).ToArray());
            return clean == new string(clean.Reverse().ToArray());
        }
    }
}
'@

Add-Type -TypeDefinition $csharp -Language CSharp

[MyTools.StringUtils]::TitleCase('hello world')  # Hello World
[MyTools.StringUtils]::Chunk('ABCDEFGH', 3)      # ABC, DEF, GH
[MyTools.StringUtils]::IsPalindrome('racecar')   # True
```

---

## 6. Async Operations

```powershell
# ใช้ async .NET methods
$task = [System.Net.Http.HttpClient]::new().GetStringAsync('https://example.com')
$html = $task.GetAwaiter().GetResult()  # block until done

# การใช้ jobs เพื่อ background work
$job = Start-Job {
    param($url)
    Invoke-WebRequest $url
} -ArgumentList 'https://example.com'

$result = Receive-Job $job -Wait -AutoRemoveJob
Write-Host "Status: $($result.StatusCode)"

# Thread jobs (PS7+, lighter than Start-Job)
$job = Start-ThreadJob {
    Start-Sleep 2
    return "Done!"
}

$result = $job | Wait-Job | Receive-Job
Write-Host $result

# Multiple parallel jobs
$urls = @('https://httpbin.org/delay/1', 'https://httpbin.org/delay/2')
$jobs = $urls | ForEach-Object {
    Start-ThreadJob -ScriptBlock {
        param($u)
        $sw = [System.Diagnostics.Stopwatch]::StartNew()
        Invoke-WebRequest $u | Out-Null
        [PSCustomObject]@{ Url=$u; Time=$sw.ElapsedMilliseconds }
    } -ArgumentList $_
}

$results = $jobs | Wait-Job | Receive-Job
$results | Format-Table
$jobs | Remove-Job
```

---

**ก่อนหน้า ← [Part 16](Part-16.md) | ต่อไป → [Part 18: WMI/CIM](Part-18.md)**
