# Part 23: Database Integration

> **ระดับ**: 🟠 Advanced | **เวลา**: ~5 ชั่วโมง

---

## 1. SQLite ด้วย .NET

```powershell
# ติดตั้ง SQLite
# Install-Module PSSQLite
# Or use System.Data.SQLite NuGet package

# สร้าง database class
class SQLiteDb {
    hidden [System.Data.SQLite.SQLiteConnection]$Conn
    
    SQLiteDb([string]$dbPath) {
        $cs = "Data Source=$dbPath;Version=3;"
        $this.Conn = [System.Data.SQLite.SQLiteConnection]::new($cs)
        $this.Conn.Open()
    }
    
    [void] Execute([string]$sql, [hashtable]$params = @{}) {
        $cmd = $this.Conn.CreateCommand()
        $cmd.CommandText = $sql
        foreach ($k in $params.Keys) {
            $cmd.Parameters.AddWithValue("@$k", $params[$k]) | Out-Null
        }
        $cmd.ExecuteNonQuery() | Out-Null
    }
    
    [System.Collections.Generic.List[hashtable]] Query([string]$sql, [hashtable]$params = @{}) {
        $cmd = $this.Conn.CreateCommand()
        $cmd.CommandText = $sql
        foreach ($k in $params.Keys) {
            $cmd.Parameters.AddWithValue("@$k", $params[$k]) | Out-Null
        }
        
        $reader  = $cmd.ExecuteReader()
        $results = [System.Collections.Generic.List[hashtable]]::new()
        
        while ($reader.Read()) {
            $row = @{}
            for ($i = 0; $i -lt $reader.FieldCount; $i++) {
                $row[$reader.GetName($i)] = if ($reader.IsDBNull($i)) { $null } else { $reader.GetValue($i) }
            }
            $results.Add($row)
        }
        $reader.Close()
        return $results
    }
    
    [object] Scalar([string]$sql, [hashtable]$params = @{}) {
        $cmd = $this.Conn.CreateCommand()
        $cmd.CommandText = $sql
        foreach ($k in $params.Keys) {
            $cmd.Parameters.AddWithValue("@$k", $params[$k]) | Out-Null
        }
        return $cmd.ExecuteScalar()
    }
    
    [void] Dispose() {
        $this.Conn.Close()
        $this.Conn.Dispose()
    }
}

# Usage
$db = [SQLiteDb]::new('myapp.db')

$db.Execute(@'
CREATE TABLE IF NOT EXISTS users (
    id      INTEGER PRIMARY KEY AUTOINCREMENT,
    name    TEXT NOT NULL,
    email   TEXT UNIQUE NOT NULL,
    created TEXT DEFAULT (datetime('now'))
)
'@)

$db.Execute('INSERT INTO users (name, email) VALUES (@name, @email)',
    @{ name='Alice'; email='alice@example.com' })

$users = $db.Query('SELECT * FROM users WHERE id > @minId', @{ minId=0 })
$users | ForEach-Object { Write-Host "$($_.id): $($_.name)" }

$count = $db.Scalar('SELECT COUNT(*) FROM users')
Write-Host "Total: $count users"

$db.Dispose()
```

---

## 2. SQL Server (ADO.NET)

```powershell
class SqlDb {
    hidden [System.Data.SqlClient.SqlConnection]$Conn
    
    SqlDb([string]$connectionString) {
        $this.Conn = [System.Data.SqlClient.SqlConnection]::new($connectionString)
        $this.Conn.Open()
    }
    
    [System.Data.DataTable] Query([string]$sql, [hashtable]$params = @{}) {
        $cmd = $this.Conn.CreateCommand()
        $cmd.CommandText = $sql
        foreach ($k in $params.Keys) {
            $cmd.Parameters.AddWithValue("@$k", $params[$k]) | Out-Null
        }
        
        $adapter = [System.Data.SqlClient.SqlDataAdapter]::new($cmd)
        $table   = [System.Data.DataTable]::new()
        $adapter.Fill($table) | Out-Null
        return $table
    }
    
    [int] Execute([string]$sql, [hashtable]$params = @{}) {
        $cmd = $this.Conn.CreateCommand()
        $cmd.CommandText = $sql
        foreach ($k in $params.Keys) {
            $cmd.Parameters.AddWithValue("@$k", $params[$k]) | Out-Null
        }
        return $cmd.ExecuteNonQuery()
    }
    
    [object] Scalar([string]$sql, [hashtable]$params = @{}) {
        $cmd = $this.Conn.CreateCommand()
        $cmd.CommandText = $sql
        foreach ($k in $params.Keys) {
            $cmd.Parameters.AddWithValue("@$k", $params[$k]) | Out-Null
        }
        return $cmd.ExecuteScalar()
    }
    
    # Bulk insert
    [void] BulkInsert([string]$table, [System.Data.DataTable]$data) {
        $bulk = [System.Data.SqlClient.SqlBulkCopy]::new($this.Conn)
        $bulk.DestinationTableName = $table
        $bulk.WriteToServer($data)
        $bulk.Close()
    }
    
    [void] Dispose() { $this.Conn.Dispose() }
}

# Connection string
$cs = 'Server=localhost;Database=myapp;Integrated Security=true'
$db = [SqlDb]::new($cs)

$users = $db.Query('SELECT TOP 10 id, name, email FROM users WHERE active = @a', @{a=1})
$users | Format-Table

$db.Dispose()
```

---

## 3. ORM-style Repository Pattern

```powershell
class UserRepository {
    hidden [SQLiteDb]$Db
    
    UserRepository([SQLiteDb]$db) {
        $this.Db = $db
        $this._EnsureSchema()
    }
    
    hidden [void] _EnsureSchema() {
        $this.Db.Execute(@'
CREATE TABLE IF NOT EXISTS users (
    id       INTEGER PRIMARY KEY AUTOINCREMENT,
    name     TEXT NOT NULL,
    email    TEXT UNIQUE NOT NULL,
    role     TEXT DEFAULT "user",
    active   INTEGER DEFAULT 1,
    created  TEXT DEFAULT (datetime("now")),
    updated  TEXT
)
'@)
    }
    
    [PSObject] GetById([int]$id) {
        $rows = $this.Db.Query('SELECT * FROM users WHERE id = @id', @{id=$id})
        if ($rows.Count -eq 0) { return $null }
        return [PSCustomObject]$rows[0]
    }
    
    [PSObject[]] GetAll([hashtable]$filters = @{}) {
        $where = @('1=1')
        $params = @{}
        
        if ($filters.role)   { $where += 'role = @role';     $params.role = $filters.role }
        if ($filters.active -ne $null) { $where += 'active = @active'; $params.active = [int]$filters.active }
        
        $sql = "SELECT * FROM users WHERE $($where -join ' AND ') ORDER BY name"
        return $this.Db.Query($sql, $params) | ForEach-Object { [PSCustomObject]$_ }
    }
    
    [PSObject] Create([hashtable]$data) {
        $this.Db.Execute(
            'INSERT INTO users (name, email, role) VALUES (@name, @email, @role)',
            @{ name=$data.name; email=$data.email; role=($data.role ?? 'user') }
        )
        $id = $this.Db.Scalar('SELECT last_insert_rowid()')
        return $this.GetById([int]$id)
    }
    
    [PSObject] Update([int]$id, [hashtable]$data) {
        $sets = @('updated = datetime("now")')
        $params = @{ id = $id }
        
        if ($data.name)   { $sets += 'name = @name';   $params.name = $data.name }
        if ($data.email)  { $sets += 'email = @email'; $params.email = $data.email }
        if ($data.role)   { $sets += 'role = @role';   $params.role = $data.role }
        if ($null -ne $data.active) { $sets += 'active = @active'; $params.active = [int]$data.active }
        
        $sql = "UPDATE users SET $($sets -join ', ') WHERE id = @id"
        $this.Db.Execute($sql, $params)
        return $this.GetById($id)
    }
    
    [bool] Delete([int]$id) {
        $rows = $this.Db.Execute('DELETE FROM users WHERE id = @id', @{id=$id})
        return $rows -gt 0
    }
}
```

---

## 4. Transactions

```powershell
class TransactionalDb {
    hidden [System.Data.IDbConnection]$Conn
    hidden [System.Data.IDbTransaction]$Transaction
    
    [void] BeginTransaction() {
        $this.Transaction = $this.Conn.BeginTransaction()
    }
    
    [void] Commit() {
        $this.Transaction.Commit()
        $this.Transaction = $null
    }
    
    [void] Rollback() {
        $this.Transaction?.Rollback()
        $this.Transaction = $null
    }
}

# Transaction usage
try {
    $db.BeginTransaction()
    
    $userId = $db.Scalar('INSERT INTO users (name,email) VALUES (@n,@e); SELECT last_insert_rowid()',
        @{n='Alice';e='alice@example.com'})
    
    $db.Execute('INSERT INTO profiles (user_id, bio) VALUES (@uid, @bio)',
        @{uid=$userId; bio='PowerShell enthusiast'})
    
    $db.Commit()
    Write-Host "User created successfully"
} catch {
    $db.Rollback()
    Write-Error "Transaction failed: $_"
}
```

---

**ก่อนหน้า ← [Part 22](Part-22.md) | ต่อไป → [Part 24: Template Engine](Part-24.md)**
