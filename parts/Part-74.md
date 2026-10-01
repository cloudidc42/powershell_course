# Part 74: Database Abstraction Layer

> **ระดับ**: 🟡 Professional | **เวลา**: ~4 ชั่วโมง

---

## 1. Repository Pattern

```powershell
# Generic repository over SQL
class DbContext {
    [string]$ConnectionString
    hidden [System.Data.SqlClient.SqlConnection]$_conn
    
    DbContext([string]$connStr) {
        $this.ConnectionString = $connStr
        $this._conn = [System.Data.SqlClient.SqlConnection]::new($connStr)
    }
    
    [void] Open()  { if ($this._conn.State -ne 'Open') { $this._conn.Open() } }
    [void] Close() { if ($this._conn.State -ne 'Closed') { $this._conn.Close() } }
    
    [System.Data.DataTable] Query([string]$sql, [hashtable]$params = @{}) {
        $this.Open()
        $cmd = $this._conn.CreateCommand()
        $cmd.CommandText = $sql
        $cmd.CommandTimeout = 30
        foreach ($kv in $params.GetEnumerator()) {
            $cmd.Parameters.AddWithValue($kv.Key, $kv.Value) | Out-Null
        }
        $adapter = [System.Data.SqlClient.SqlDataAdapter]::new($cmd)
        $table   = [System.Data.DataTable]::new()
        $adapter.Fill($table) | Out-Null
        return $table
    }
    
    [int] Execute([string]$sql, [hashtable]$params = @{}) {
        $this.Open()
        $cmd = $this._conn.CreateCommand()
        $cmd.CommandText = $sql
        foreach ($kv in $params.GetEnumerator()) {
            $cmd.Parameters.AddWithValue($kv.Key, $kv.Value) | Out-Null
        }
        return $cmd.ExecuteNonQuery()
    }
    
    [object] Scalar([string]$sql, [hashtable]$params = @{}) {
        $this.Open()
        $cmd = $this._conn.CreateCommand()
        $cmd.CommandText = $sql
        foreach ($kv in $params.GetEnumerator()) {
            $cmd.Parameters.AddWithValue($kv.Key, $kv.Value) | Out-Null
        }
        return $cmd.ExecuteScalar()
    }
    
    [object] InTransaction([scriptblock]$work) {
        $this.Open()
        $tx = $this._conn.BeginTransaction()
        $cmd = $this._conn.CreateCommand()
        $cmd.Transaction = $tx
        try {
            $result = & $work $this
            $tx.Commit()
            return $result
        } catch {
            $tx.Rollback()
            throw
        }
    }
    
    [void] Dispose() { $this._conn.Dispose() }
}

# Typed repository
class UserRepository {
    hidden [DbContext]$_db
    
    UserRepository([DbContext]$db) { $this._db = $db }
    
    [PSCustomObject] GetById([int]$id) {
        $table = $this._db.Query('SELECT * FROM Users WHERE Id = @Id', @{ '@Id'=$id })
        if ($table.Rows.Count -eq 0) { return $null }
        $row = $table.Rows[0]
        return [PSCustomObject]@{
            Id        = $row.Id
            Name      = $row.Name
            Email     = $row.Email
            CreatedAt = $row.CreatedAt
        }
    }
    
    [PSCustomObject[]] Search([string]$term, [int]$limit = 50) {
        $table = $this._db.Query(
            'SELECT TOP (@Limit) * FROM Users WHERE Name LIKE @Term OR Email LIKE @Term',
            @{ '@Term'="%$term%"; '@Limit'=$limit }
        )
        return $table.Rows | ForEach-Object {
            [PSCustomObject]@{ Id=$_.Id; Name=$_.Name; Email=$_.Email }
        }
    }
    
    [int] Create([string]$name, [string]$email) {
        return [int]$this._db.Scalar(
            'INSERT INTO Users (Name, Email, CreatedAt) OUTPUT INSERTED.Id VALUES (@Name, @Email, GETDATE())',
            @{ '@Name'=$name; '@Email'=$email }
        )
    }
    
    [bool] Update([int]$id, [hashtable]$fields) {
        $setClauses = ($fields.Keys | ForEach-Object { "$_ = @$_" }) -join ', '
        $params     = @{ '@Id'=$id } + ($fields.GetEnumerator() | ForEach-Object {
            @{ "@$($_.Key)" = $_.Value }
        } | ForEach-Object { $_ })
        $affected = $this._db.Execute("UPDATE Users SET $setClauses WHERE Id = @Id", $params)
        return $affected -gt 0
    }
    
    [bool] Delete([int]$id) {
        return $this._db.Execute('DELETE FROM Users WHERE Id = @Id', @{ '@Id'=$id }) -gt 0
    }
}

$db   = [DbContext]::new($env:DB_CONNECTION_STRING)
$repo = [UserRepository]::new($db)

$user    = $repo.GetById(1)
$results = $repo.Search('alice')
$newId   = $repo.Create('Bob Smith', 'bob@example.com')
$repo.Update($newId, @{ Name='Robert Smith' })
```

---

## 2. SQLite เบา (Embedded DB)

```powershell
# SQLite via System.Data.SQLite
# Download: Install-Package System.Data.SQLite.Core

$sqlitePath = Join-Path $env:TEMP 'myapp.db'
$connStr    = "Data Source=$sqlitePath;Version=3;"

function Invoke-SQLite {
    param([string]$Sql, [hashtable]$Params = @{}, [switch]$Scalar)
    
    Add-Type -Path 'C:\tools\SQLite\System.Data.SQLite.dll'
    $conn = [System.Data.SQLite.SQLiteConnection]::new($connStr)
    $conn.Open()
    
    try {
        $cmd = $conn.CreateCommand()
        $cmd.CommandText = $Sql
        foreach ($kv in $Params.GetEnumerator()) {
            $cmd.Parameters.AddWithValue($kv.Key, $kv.Value) | Out-Null
        }
        
        if ($Scalar) { return $cmd.ExecuteScalar() }
        
        $adapter = [System.Data.SQLite.SQLiteDataAdapter]::new($cmd)
        $table   = [System.Data.DataTable]::new()
        $adapter.Fill($table) | Out-Null
        return $table
    } finally {
        $conn.Close()
    }
}

# Create schema
Invoke-SQLite @'
CREATE TABLE IF NOT EXISTS logs (
    id        INTEGER PRIMARY KEY AUTOINCREMENT,
    ts        DATETIME DEFAULT CURRENT_TIMESTAMP,
    level     TEXT NOT NULL,
    message   TEXT NOT NULL,
    source    TEXT
);
'@

# Insert
Invoke-SQLite 'INSERT INTO logs (level, message, source) VALUES (@l, @m, @s)' `
    -Params @{ '@l'='INFO'; '@m'='App started'; '@s'='main' }

# Query
$logs = Invoke-SQLite 'SELECT * FROM logs ORDER BY ts DESC LIMIT 100'
$logs.Rows | ForEach-Object { [PSCustomObject]@{ Time=$_.ts; Level=$_.level; Msg=$_.message } }
```

---

**ก่อนหน้า ← [Part 73](Part-73.md) | ต่อไป → [Part 75: Async Programming](Part-75.md)**
