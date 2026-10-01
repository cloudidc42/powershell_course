# Part 38: Binary Modules และ C# Integration

> **ระดับ**: 🟠 Advanced | **เวลา**: ~4 ชั่วขนึ่ง

---

## 1. สร้าง Binary Module ด้วย C#

```csharp
// MyModule/MyModule.cs
using System.Management.Automation;
using System.Management.Automation.Runspaces;

namespace MyModule
{
    [Cmdlet(VerbsCommon.Get, "RandomPassword")]
    [OutputType(typeof(string))]
    public class GetRandomPasswordCmdlet : PSCmdlet
    {
        [Parameter(Position = 0)]
        [ValidateRange(8, 128)]
        public int Length { get; set; } = 16;
        
        [Parameter]
        public SwitchParameter IncludeSpecial { get; set; }
        
        protected override void ProcessRecord()
        {
            const string lower   = "abcdefghijklmnopqrstuvwxyz";
            const string upper   = "ABCDEFGHIJKLMNOPQRSTUVWXYZ";
            const string digits  = "0123456789";
            const string special = "!@#$%^&*()-_=+";
            
            var pool = lower + upper + digits;
            if (IncludeSpecial) pool += special;
            
            var rng = new System.Security.Cryptography.RNGCryptoServiceProvider();
            var bytes = new byte[Length];
            rng.GetBytes(bytes);
            var password = new char[Length];
            for (int i = 0; i < Length; i++)
                password[i] = pool[bytes[i] % pool.Length];
            
            WriteObject(new string(password));
        }
    }
}
```

```xml
<!-- MyModule/MyModule.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net6.0</TargetFramework>
    <AssemblyName>MyModule</AssemblyName>
    <Nullable>enable</Nullable>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="System.Management.Automation" Version="7.3.6" />
  </ItemGroup>
</Project>
```

```powershell
# Build และ import
dotnet build MyModule.csproj -c Release -o .\bin
Import-Module .\bin\MyModule.dll

Get-RandomPassword -Length 20 -IncludeSpecial
```

---

## 2. Cmdlet Lifecycle

```csharp
[Cmdlet(VerbsData.Export, "CsvFast")]
public class ExportCsvFastCmdlet : PSCmdlet
{
    [Parameter(Mandatory=true, ValueFromPipeline=true)]
    public PSObject InputObject { get; set; }
    
    [Parameter(Mandatory=true, Position=0)]
    public string Path { get; set; }
    
    private System.IO.StreamWriter _writer;
    private bool _headerWritten;
    private List<string> _properties;
    
    protected override void BeginProcessing()
    {
        _writer = new System.IO.StreamWriter(Path, false, System.Text.Encoding.UTF8);
        _headerWritten = false;
    }
    
    protected override void ProcessRecord()
    {
        if (!_headerWritten)
        {
            _properties = InputObject.Properties.Select(p => p.Name).ToList();
            _writer.WriteLine(string.Join(",", _properties.Select(EscapeCsv)));
            _headerWritten = true;
        }
        _writer.WriteLine(string.Join(",", _properties.Select(p =>
            EscapeCsv(InputObject.Properties[p]?.Value?.ToString() ?? "")
        )));
    }
    
    protected override void EndProcessing()
    {
        _writer?.Flush();
        _writer?.Close();
        WriteObject($"Exported to {Path}");
    }
    
    protected override void StopProcessing()
    {
        _writer?.Close();
    }
    
    private static string EscapeCsv(string value)
        => value.Contains(',') || value.Contains('"') || value.Contains('\n')
            ? $"\"{ value.Replace("\"", "\"\"") }\""
            : value;
}
```

---

## 3. Dynamic Parameters

```csharp
[Cmdlet(VerbsCommon.Get, "DatabaseObject")]
public class GetDatabaseObjectCmdlet : PSCmdlet, IDynamicParameters
{
    [Parameter(Mandatory=true)]
    public string TableName { get; set; }
    
    private RuntimeDefinedParameterDictionary _dynamicParams;
    
    public object GetDynamicParameters()
    {
        _dynamicParams = new RuntimeDefinedParameterDictionary();
        
        // Add Id parameter only for specific tables
        var idAttr = new ParameterAttribute { Mandatory = false };
        var idColl = new Collection<Attribute> { idAttr };
        _dynamicParams.Add("Id", new RuntimeDefinedParameter("Id", typeof(int), idColl));
        
        return _dynamicParams;
    }
    
    protected override void ProcessRecord()
    {
        int? id = null;
        if (_dynamicParams.ContainsKey("Id") && _dynamicParams["Id"].IsSet)
            id = (int)_dynamicParams["Id"].Value;
        
        WriteObject($"Query {TableName} Id={id}");
    }
}
```

---

## 4. สร้าง .psd1 สำหรับ Binary Module

```powershell
# MyModule.psd1
@{
    ModuleVersion     = '1.0.0'
    GUID              = 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
    Author            = 'Me'
    Description       = 'High performance cmdlets'
    RootModule        = 'MyModule.dll'
    PowerShellVersion = '7.0'
    RequiredAssemblies = @('MyModule.dll')
    CmdletsToExport   = @('Get-RandomPassword', 'Export-CsvFast', 'Get-DatabaseObject')
    FunctionsToExport = @()
    AliasesToExport   = @()
    PrivateData       = @{
        PSData = @{
            Tags        = @('Performance', 'Utility')
            ProjectUri  = 'https://github.com/user/mymodule'
            LicenseUri  = 'https://github.com/user/mymodule/blob/main/LICENSE'
        }
    }
}
```

---

**ก่อนหน้า ← [Part 37](Part-37.md) | ต่อไป → [Part 39: Performance Optimization](Part-39.md)**
