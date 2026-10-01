# Part 77: Machine Learning Integration

> **ระดับ**: 🟠 World-Class | **เวลา**: ~5 ชั่วโมง

---

## 1. Azure ML Integration

```powershell
Install-Module Az.MachineLearning -Force

# Connect
Connect-AzAccount -ServicePrincipal -Credential $cred -TenantId $env:AZURE_TENANT_ID

$workspace = Get-AzMlWorkspace -ResourceGroupName 'mlrg' -Name 'myml'

# List models
Get-AzMlModel -Workspace $workspace | Select-Object Name, Version, CreatedTime | Format-Table

# Deploy model as endpoint
$deployment = New-AzMlOnlineEndpoint `
    -ResourceGroupName 'mlrg' `
    -WorkspaceName     'myml' `
    -Name              'fraud-detector' `
    -AuthMode          'Key'

# Invoke ML endpoint for prediction
function Invoke-MLEndpoint {
    param(
        [string]$EndpointUrl,
        [string]$ApiKey,
        [object[]]$InputData
    )
    
    $payload = @{
        input_data = @{
            columns = ($InputData[0].PSObject.Properties.Name)
            data    = $InputData | ForEach-Object {
                $_.PSObject.Properties.Value
            }
        }
    } | ConvertTo-Json -Depth 10
    
    $result = Invoke-RestMethod `
        -Uri     $EndpointUrl `
        -Method  POST `
        -Headers @{ Authorization="Bearer $ApiKey"; 'Content-Type'='application/json' } `
        -Body    $payload
    
    return $result
}

$transactions = @(
    @{ amount=150.00; merchant='Amazon';   hour=14; dayOfWeek=2 }
    @{ amount=9999.99; merchant='Unknown'; hour=3;  dayOfWeek=6 }
)

$predictions = Invoke-MLEndpoint `
    -EndpointUrl $env:ML_ENDPOINT_URL `
    -ApiKey      $env:ML_ENDPOINT_KEY `
    -InputData   $transactions

$transactions | ForEach-Object { $i=0 } {
    [PSCustomObject]@{
        Amount      = $_.amount
        Merchant    = $_.merchant
        FraudScore  = $predictions[$i]
        Flagged     = $predictions[$i] -gt 0.7
    }
    $i++
} | Format-Table
```

---

## 2. OpenAI / Claude API Integration

```powershell
# OpenAI ChatGPT
function Invoke-ChatGPT {
    param(
        [string]$Prompt,
        [string]$Model   = 'gpt-4',
        [double]$Temp    = 0.7,
        [int]$MaxTokens  = 1000,
        [string[]]$SystemMessages = @('You are a helpful PowerShell expert.')
    )
    
    $messages = @(
        $SystemMessages | ForEach-Object { @{ role='system'; content=$_ } }
        @{ role='user'; content=$Prompt }
    )
    
    $payload = @{
        model       = $Model
        messages    = $messages
        temperature = $Temp
        max_tokens  = $MaxTokens
    } | ConvertTo-Json -Depth 10
    
    $response = Invoke-RestMethod `
        -Uri     'https://api.openai.com/v1/chat/completions' `
        -Method  POST `
        -Headers @{ Authorization="Bearer $env:OPENAI_API_KEY"; 'Content-Type'='application/json' } `
        -Body    $payload
    
    return $response.choices[0].message.content
}

# Anthropic Claude API
function Invoke-ClaudeAPI {
    param(
        [string]$Prompt,
        [string]$Model     = 'claude-sonnet-4-6',
        [int]$MaxTokens    = 1024,
        [string]$System    = ''
    )
    
    $body = @{
        model      = $Model
        max_tokens = $MaxTokens
        messages   = @(@{ role='user'; content=$Prompt })
    }
    if ($System) { $body['system'] = $System }
    
    $response = Invoke-RestMethod `
        -Uri     'https://api.anthropic.com/v1/messages' `
        -Method  POST `
        -Headers @{
            'x-api-key'         = $env:ANTHROPIC_API_KEY
            'anthropic-version' = '2023-06-01'
            'Content-Type'      = 'application/json'
        } `
        -Body ($body | ConvertTo-Json -Depth 10)
    
    return $response.content[0].text
}

# PowerShell code generator
$script = Invoke-ClaudeAPI `
    -Prompt 'Write a PowerShell function to get the top 5 largest files in a directory recursively' `
    -System 'You are an expert PowerShell developer. Respond with only runnable PowerShell code.'

Write-Host $script
```

---

## 3. ML.NET Integration

```powershell
# ML.NET for local model inference
Add-Type -Path 'Microsoft.ML.dll'

# Sentiment analysis with pre-trained model
function Test-Sentiment {
    param([string[]]$Texts)
    
    $mlContext   = [Microsoft.ML.MLContext]::new()
    $model       = $mlContext.Model.Load('sentiment-model.zip', [ref]$null)
    $predEngine  = $mlContext.Model.CreatePredictionEngine[SentimentInput, SentimentOutput]($model)
    
    $Texts | ForEach-Object {
        $input  = [SentimentInput]@{ SentimentText = $_ }
        $pred   = $predEngine.Predict($input)
        [PSCustomObject]@{
            Text      = $_.Substring(0, [Math]::Min(50, $_.Length))
            Positive  = $pred.Prediction
            Confidence= [math]::Round($pred.Probability, 3)
        }
    }
}

Test-Sentiment @(
    'This product is absolutely amazing!'
    'Terrible experience, never again.'
    'The service was okay, nothing special.'
) | Format-Table

# Time-series anomaly detection
function Find-Anomalies {
    param([double[]]$Values, [int]$Period = 12, [double]$Confidence = 0.95)
    
    $mlContext = [Microsoft.ML.MLContext]::new()
    $data = $Values | ForEach-Object { @{ value=[float]$_ } } | ForEach-Object { [PSCustomObject]$_ }
    $dataView = $mlContext.Data.LoadFromEnumerable($data)
    
    $pipeline = $mlContext.Transforms.DetectIidSpike(
        outputColumnName='Prediction',
        inputColumnName='value',
        confidence=$Confidence * 100,
        pvalueHistoryLength=[int]($Values.Count / 4)
    )
    
    $model     = $pipeline.Fit($dataView)
    $transformed = $model.Transform($dataView)
    $predictions  = $mlContext.Data.CreateEnumerable[AnomalyPrediction]($transformed, $false)
    
    $i = 0
    return $predictions | Where-Object { $_.Prediction[0] -eq 1 } | ForEach-Object {
        @{ Index=$i; Value=$Values[$i]; Score=$_.Prediction[1] }
        $i++
    }
}
```

---

## 4. Vector Search และ Embeddings

```powershell
# Semantic search with embeddings
function Get-Embedding {
    param([string]$Text, [string]$Model = 'text-embedding-3-small')
    
    $response = Invoke-RestMethod `
        -Uri     'https://api.openai.com/v1/embeddings' `
        -Method  POST `
        -Headers @{ Authorization="Bearer $env:OPENAI_API_KEY"; 'Content-Type'='application/json' } `
        -Body    (@{ model=$Model; input=$Text } | ConvertTo-Json)
    
    return $response.data[0].embedding
}

function Get-CosineSimilarity {
    param([double[]]$A, [double[]]$B)
    
    $dot    = 0.0
    $magA   = 0.0
    $magB   = 0.0
    
    for ($i = 0; $i -lt $A.Count; $i++) {
        $dot  += $A[$i] * $B[$i]
        $magA += $A[$i] * $A[$i]
        $magB += $B[$i] * $B[$i]
    }
    
    return $dot / ([Math]::Sqrt($magA) * [Math]::Sqrt($magB))
}

# Semantic document search
$docs = @(
    'PowerShell is a task automation framework'
    'Kubernetes manages containerized applications'
    'Machine learning enables pattern recognition'
    'Azure provides cloud computing services'
)

# Index documents
$indexed = $docs | ForEach-Object {
    @{ text=$_; embedding=(Get-Embedding $_) }
}

# Search
$query     = 'container orchestration'
$queryEmb  = Get-Embedding $query

$results   = $indexed | ForEach-Object {
    [PSCustomObject]@{
        Text       = $_.text
        Similarity = Get-CosineSimilarity $queryEmb $_.embedding
    }
} | Sort-Object Similarity -Descending

Write-Host "Top results for '$query':"
$results | Select-Object -First 3 | Format-Table
```

---

**ก่อนหน้า ← [Part 76](Part-76.md) | ต่อไป → [Part 78: GraphQL API Integration](Part-78.md)**
