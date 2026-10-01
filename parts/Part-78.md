# Part 78: GraphQL API Integration

> **ระดับ**: 🟠 World-Class | **เวลย**: ~4 ชั่วโมง

---

## 1. GraphQL Client

```powershell
class GraphQLClient {
    [string]$Endpoint
    [hashtable]$Headers
    
    GraphQLClient([string]$endpoint, [hashtable]$headers = @{}) {
        $this.Endpoint = $endpoint
        $this.Headers  = @{ 'Content-Type'='application/json' } + $headers
    }
    
    [object] Query([string]$query, [hashtable]$variables = @{}) {
        $body = @{ query=$query } + (if ($variables.Count -gt 0) { @{variables=$variables} } else { @{} })
        
        $response = Invoke-RestMethod `
            -Uri     $this.Endpoint `
            -Method  POST `
            -Headers $this.Headers `
            -Body    ($body | ConvertTo-Json -Depth 10)
        
        if ($response.errors) {
            $errMsg = ($response.errors | ForEach-Object { $_.message }) -join '; '
            throw "GraphQL Error: $errMsg"
        }
        
        return $response.data
    }
    
    [object] Mutate([string]$mutation, [hashtable]$variables = @{}) {
        return $this.Query($mutation, $variables)
    }
}

$gql = [GraphQLClient]::new(
    'https://api.example.com/graphql',
    @{ Authorization = "Bearer $env:API_TOKEN" }
)

# Query
$result = $gql.Query(@'
    query GetUsers($limit: Int, $offset: Int) {
        users(limit: $limit, offset: $offset) {
            id
            name
            email
            createdAt
            orders {
                id
                total
                status
            }
        }
        usersAggregate {
            count
        }
    }
'@, @{ limit=20; offset=0 })

$result.users | Select-Object id, name, email, @{n='OrderCount';e={$_.orders.Count}} | Format-Table
```

---

## 2. GitHub GraphQL API

```powershell
$github = [GraphQLClient]::new(
    'https://api.github.com/graphql',
    @{ Authorization = "Bearer $env:GITHUB_TOKEN" }
)

# Get repository info with PRs
$repoData = $github.Query(@'
    query RepoInfo($owner: String!, $repo: String!) {
        repository(owner: $owner, name: $repo) {
            name
            description
            stargazerCount
            forkCount
            issues(states: [OPEN]) { totalCount }
            pullRequests(states: [OPEN]) { totalCount }
            defaultBranchRef {
                name
                target {
                    ... on Commit {
                        history(first: 5) {
                            nodes {
                                message
                                committedDate
                                author { name email }
                            }
                        }
                    }
                }
            }
        }
    }
'@, @{ owner='microsoft'; repo='PowerShell' })

$repo = $repoData.repository
Write-Host "$($repo.name): $($repo.stargazerCount) stars, $($repo.forkCount) forks"
Write-Host "Open PRs: $($repo.pullRequests.totalCount), Issues: $($repo.issues.totalCount)"
$repo.defaultBranchRef.target.history.nodes | ForEach-Object {
    "- $($_.committedDate.Substring(0,10)) $($_.message.Split("`n")[0].Substring(0,[Math]::Min(60,$_.message.Length)))"
}

# Search repositories
$search = $github.Query(@'
    query SearchRepos($q: String!) {
        search(query: $q, type: REPOSITORY, first: 10) {
            repositoryCount
            nodes {
                ... on Repository {
                    nameWithOwner
                    stargazerCount
                    description
                    primaryLanguage { name }
                }
            }
        }
    }
'@, @{ q='powershell security language:powershell stars:>100' })

$search.search.nodes | Sort-Object stargazerCount -Descending | Format-Table nameWithOwner, stargazerCount, @{n='Language';e={$_.primaryLanguage.name}}
```

---

## 3. GraphQL Subscription (WebSocket)

```powershell
# GraphQL subscriptions via WebSocket
function Connect-GraphQLSubscription {
    param([string]$WsEndpoint, [string]$Token, [string]$Subscription, [scriptblock]$OnData)
    
    $ws = [System.Net.WebSockets.ClientWebSocket]::new()
    $ws.Options.SetRequestHeader('Authorization', "Bearer $Token")
    
    $uri    = [System.Uri]::new($WsEndpoint)
    $cancel = [System.Threading.CancellationToken]::None
    
    $ws.ConnectAsync($uri, $cancel).Wait()
    
    # GraphQL WS protocol: connection_init
    $initMsg = '{"type":"connection_init","payload":{}}'
    $bytes   = [System.Text.Encoding]::UTF8.GetBytes($initMsg)
    $ws.SendAsync($bytes, [System.Net.WebSockets.WebSocketMessageType]::Text, $true, $cancel).Wait()
    
    # Subscribe
    $subMsg = @{
        id      = '1'
        type    = 'start'
        payload = @{ query = $Subscription }
    } | ConvertTo-Json
    $bytes = [System.Text.Encoding]::UTF8.GetBytes($subMsg)
    $ws.SendAsync($bytes, [System.Net.WebSockets.WebSocketMessageType]::Text, $true, $cancel).Wait()
    
    # Receive loop
    $buffer = [byte[]]::new(4096)
    while ($ws.State -eq 'Open') {
        $segment = [System.ArraySegment[byte]]::new($buffer)
        $result  = $ws.ReceiveAsync($segment, $cancel).GetAwaiter().GetResult()
        
        if ($result.MessageType -eq 'Close') { break }
        
        $json = [System.Text.Encoding]::UTF8.GetString($buffer, 0, $result.Count)
        $msg  = $json | ConvertFrom-Json
        
        if ($msg.type -eq 'data' -and $msg.payload.data) {
            & $OnData $msg.payload.data
        }
    }
    
    $ws.Dispose()
}

Connect-GraphQLSubscription `
    -WsEndpoint  'wss://api.example.com/graphql' `
    -Token       $env:API_TOKEN `
    -Subscription 'subscription { orderCreated { id amount customerId } }' `
    -OnData {
        param($data)
        Write-Host "New order: #$($data.orderCreated.id) - $$($data.orderCreated.amount)" -ForegroundColor Green
    }
```

---

**ก่อนหน้า ← [Part 77](Part-77.md) | ต่อไป → [Part 79: DevSecOps Automation](Part-79.md)**
