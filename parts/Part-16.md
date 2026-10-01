# Part 16: Classes และ OOP ใน PowerShell

> **ระดับ**: 🟡 Intermediate | **เวลา**: ~4 ชั่วโมง

---

## 1. Class Basics

```powershell
class Animal {
    # Properties
    [string]$Name
    [int]$Age
    [string]$Species
    
    # Static property
    static [int]$Count = 0
    
    # Hidden property
    hidden [string]$_internalId
    
    # Constructor
    Animal([string]$name, [int]$age, [string]$species) {
        $this.Name    = $name
        $this.Age     = $age
        $this.Species = $species
        $this._internalId = [guid]::NewGuid().ToString()
        [Animal]::Count++
    }
    
    # Method
    [string] Speak() {
        return "$($this.Name) makes a sound"
    }
    
    [string] ToString() {
        return "$($this.Species): $($this.Name) (Age: $($this.Age))"
    }
    
    # Static method
    static [Animal] Create([string]$name, [int]$age, [string]$species) {
        return [Animal]::new($name, $age, $species)
    }
}

# สร้าง instance
$cat = [Animal]::new('Whiskers', 3, 'Cat')
$dog = [Animal]::new('Rex', 5, 'Dog')

$cat.Name     # Whiskers
$cat.Speak()  # Whiskers makes a sound
$cat.ToString()  # Cat: Whiskers (Age: 3)
[Animal]::Count  # 2

# Static factory
$bird = [Animal]::Create('Tweety', 2, 'Bird')
```

---

## 2. Inheritance

```powershell
class Dog : Animal {
    [string]$Breed
    [bool]$IsGoodBoy = $true
    
    # Constructor calls parent with base()
    Dog([string]$name, [int]$age, [string]$breed) : base($name, $age, 'Dog') {
        $this.Breed = $breed
    }
    
    # Override method
    [string] Speak() {
        return "$($this.Name) barks: Woof!"
    }
    
    [void] Fetch([string]$item) {
        Write-Host "$($this.Name) fetches the $item!"
    }
    
    [string] ToString() {
        return "Dog: $($this.Name) ($($this.Breed))"
    }
}

class GuideDog : Dog {
    [string]$Handler
    
    GuideDog([string]$name, [string]$breed, [string]$handler) : base($name, 2, $breed) {
        $this.Handler = $handler
    }
    
    [string] Speak() {
        return "$($this.Name) guides $($this.Handler) quietly"
    }
}

$dog = [Dog]::new('Rex', 5, 'German Shepherd')
$dog.Speak()    # Rex barks: Woof!
$dog.Fetch('ball')

$guide = [GuideDog]::new('Buddy', 'Labrador', 'John')
$guide.Speak()  # Buddy guides John quietly

# Type checks
$guide -is [GuideDog]  # True
$guide -is [Dog]       # True
$guide -is [Animal]    # True
```

---

## 3. Interfaces (ผ่าน .NET)

```powershell
# ใช้ .NET interfaces
class MyComparer : System.Collections.Generic.IComparer[int] {
    [int] Compare([int]$x, [int]$y) {
        if ($x -lt $y) { return -1 }
        if ($x -gt $y) { return  1 }
        return 0
    }
}

# IDisposable
class DatabaseConnection : System.IDisposable {
    [string]$ConnectionString
    hidden [object]$_connection
    hidden [bool]$_disposed = $false
    
    DatabaseConnection([string]$cs) {
        $this.ConnectionString = $cs
        # $this._connection = Open-Connection $cs
        Write-Verbose "Opened connection to $cs"
    }
    
    [void] Execute([string]$query) {
        if ($this._disposed) { throw 'Connection is disposed' }
        Write-Host "Executing: $query"
    }
    
    [void] Dispose() {
        if (!$this._disposed) {
            # $this._connection.Close()
            $this._disposed = $true
            Write-Verbose "Connection closed"
        }
    }
}

# ใช้กับ using
$db = [DatabaseConnection]::new('Server=localhost;Database=test')
try {
    $db.Execute('SELECT 1')
} finally {
    $db.Dispose()
}
```

---

## 4. Properties with Validation

```powershell
class Person {
    hidden [string]$_name
    hidden [int]$_age
    hidden [string]$_email
    
    # Property with getter/setter via method pattern
    [string] GetName() { return $this._name }
    [void] SetName([string]$value) {
        if ([string]::IsNullOrWhiteSpace($value)) {
            throw [System.ArgumentException]::new('Name cannot be empty')
        }
        $this._name = $value.Trim()
    }
    
    [int] GetAge() { return $this._age }
    [void] SetAge([int]$value) {
        if ($value -lt 0 -or $value -gt 150) {
            throw [System.ArgumentOutOfRangeException]::new('age', 'Age must be 0-150')
        }
        $this._age = $value
    }
    
    [string] GetEmail() { return $this._email }
    [void] SetEmail([string]$value) {
        if ($value -notmatch '^[^@]+@[^@]+\.[^@]+$') {
            throw [System.ArgumentException]::new('Invalid email format')
        }
        $this._email = $value.ToLower()
    }
    
    Person([string]$name, [int]$age, [string]$email) {
        $this.SetName($name)
        $this.SetAge($age)
        $this.SetEmail($email)
    }
    
    [string] ToString() {
        return "Person{name=$($this._name), age=$($this._age)}"
    }
}

$p = [Person]::new('Alice', 30, 'alice@example.com')
$p.GetName()    # Alice
$p.SetAge(31)

try {
    $p.SetEmail('not-valid')
} catch {
    Write-Host "Error: $_"
}
```

---

## 5. Design Patterns

```powershell
# Singleton Pattern
class AppConfig {
    static hidden [AppConfig]$_instance = $null
    
    [hashtable]$Settings = @{}
    
    hidden AppConfig() { }
    
    static [AppConfig] GetInstance() {
        if ($null -eq [AppConfig]::_instance) {
            [AppConfig]::_instance = [AppConfig]::new()
        }
        return [AppConfig]::_instance
    }
    
    [void] Load([string]$path) {
        if (Test-Path $path) {
            $this.Settings = Get-Content $path -Raw | ConvertFrom-Json -AsHashtable
        }
    }
}

$cfg1 = [AppConfig]::GetInstance()
$cfg2 = [AppConfig]::GetInstance()
[object]::ReferenceEquals($cfg1, $cfg2)  # True - same instance

# Builder Pattern
class QueryBuilder {
    hidden [string]$_table = ''
    hidden [System.Collections.Generic.List[string]]$_where
    hidden [System.Collections.Generic.List[string]]$_select
    hidden [string]$_orderBy = ''
    hidden [int]$_limit = 0
    
    QueryBuilder() {
        $this._where  = [System.Collections.Generic.List[string]]::new()
        $this._select = [System.Collections.Generic.List[string]]::new()
    }
    
    [QueryBuilder] From([string]$table) {
        $this._table = $table; return $this
    }
    
    [QueryBuilder] Select([string[]]$cols) {
        $cols | ForEach-Object { $this._select.Add($_) }
        return $this
    }
    
    [QueryBuilder] Where([string]$condition) {
        $this._where.Add($condition); return $this
    }
    
    [QueryBuilder] OrderBy([string]$col) {
        $this._orderBy = $col; return $this
    }
    
    [QueryBuilder] Limit([int]$n) {
        $this._limit = $n; return $this
    }
    
    [string] Build() {
        $cols = if ($this._select.Count) { $this._select -join ', ' } else { '*' }
        $sql  = "SELECT $cols FROM $($this._table)"
        if ($this._where.Count) { $sql += " WHERE " + ($this._where -join ' AND ') }
        if ($this._orderBy) { $sql += " ORDER BY $($this._orderBy)" }
        if ($this._limit -gt 0) { $sql += " LIMIT $($this._limit)" }
        return $sql
    }
}

$query = [QueryBuilder]::new()
    .From('users')
    .Select(@('id','name','email'))
    .Where("age > 18")
    .Where("active = 1")
    .OrderBy('name')
    .Limit(10)
    .Build()

Write-Host $query
# SELECT id, name, email FROM users WHERE age > 18 AND active = 1 ORDER BY name LIMIT 10
```

---

**ก่อนหน้า ← [Part 15](Part-15.md) | ต่อไป → [Part 17: .NET Integration](Part-17.md)**
