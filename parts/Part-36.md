# Part 36: AWS ด้วย PowerShell

> **ระดับ**: 🟠 Advanced | **เวลา**: ~4 ชั่วขนึ่ง

---

## 1. Setup AWS Tools

```powershell
# ติดตั้ง
Install-Module AWSPowerShell.NetCore -Force
# หรือติดตั้งทีละ service:
Install-Module AWS.Tools.EC2, AWS.Tools.S3, AWS.Tools.Lambda, AWS.Tools.ECS -Force

# Configure credentials
Set-AWSCredential -AccessKey 'AKIAIOSFODNN7EXAMPLE' -SecretKey 'wJalrXUtnFEMI...' -StoreAs default

# Profile-based
Initialize-AWSDefaultConfiguration -ProfileName 'prod' -Region 'ap-southeast-1'

# IAM Role ใน EC2 instance (ไม่ต้องกำหนด key)
Set-AWSCredential -InstanceProfile
```

---

## 2. EC2 Management

```powershell
# List instances
Get-EC2Instance | ForEach-Object {
    $inst = $_.Instances[0]
    [PSCustomObject]@{
        Id    = $inst.InstanceId
        Type  = $inst.InstanceType
        State = $inst.State.Name
        Name  = ($inst.Tags | Where-Object Key -eq 'Name').Value
        IP    = $inst.PublicIpAddress
    }
}

# Start/Stop
Start-EC2Instance -InstanceId 'i-0123456789abcdef0'
Stop-EC2Instance  -InstanceId 'i-0123456789abcdef0' -Force

# Launch instance
$sg = New-EC2SecurityGroup -GroupName 'web-sg' -Description 'Web server SG' -VpcId 'vpc-xxx'
Grant-EC2SecurityGroupIngress -GroupId $sg -IpPermission @(
    @{IpProtocol='tcp'; FromPort=80;  ToPort=80;  IpRanges='0.0.0.0/0'},
    @{IpProtocol='tcp'; FromPort=443; ToPort=443; IpRanges='0.0.0.0/0'}
)

$inst = New-EC2Instance `
    -ImageId    'ami-0c55b159cbfafe1f0' `
    -InstanceType 't3.micro' `
    -SecurityGroupId $sg `
    -KeyName 'my-keypair' `
    -MinCount 1 -MaxCount 1

$instanceId = $inst.Instances[0].InstanceId
Write-Host "Launched: $instanceId"

# Wait for running
Wait-EC2State -InstanceId $instanceId -DesiredState Running
```

---

## 3. S3 Operations

```powershell
# Buckets
Get-S3Bucket
New-S3Bucket -BucketName 'my-unique-bucket-12345' -Region 'ap-southeast-1'

# Upload files
Write-S3Object -BucketName 'my-bucket' -File 'C:\report.pdf' -Key 'reports/2024/report.pdf'

# Upload folder
Write-S3Object -BucketName 'my-bucket' -Folder 'C:\App\dist' -KeyPrefix 'app/v2/' -Recurse

# Download
Read-S3Object -BucketName 'my-bucket' -Key 'reports/2024/report.pdf' -File 'C:\Download\report.pdf'

# List objects
Get-S3Object -BucketName 'my-bucket' -Prefix 'reports/' |
    Select-Object Key, Size, LastModified | Format-Table -AutoSize

# Presigned URL
$url = Get-S3PreSignedURL `
    -BucketName 'my-bucket' `
    -Key 'reports/2024/report.pdf' `
    -Expires (Get-Date).AddHours(24) `
    -Verb GET
Write-Host $url

# Sync (like aws s3 sync)
Sync-S3Objects -BucketName 'my-bucket' -LocalFolder 'C:\App\dist' -KeyPrefix 'app/'
```

---

## 4. Lambda และ ECS

```powershell
# Lambda
Get-LMFunctionList | Select-Object FunctionName, Runtime, LastModified

# Invoke Lambda
$payload = @{action='process'; id=42} | ConvertTo-Json
$result  = Invoke-LMFunction -FunctionName 'my-processor' -Payload $payload
$output  = [System.IO.StreamReader]::new($result.Payload).ReadToEnd() | ConvertFrom-Json
Write-Host "Result: $($output.status)"

# Deploy Lambda (zip)
Compress-Archive -Path '.\lambda\*' -DestinationPath 'function.zip' -Force
Update-LMFunctionCode -FunctionName 'my-processor' -ZipFilename 'function.zip'

# ECS Tasks
Get-ECSClusterList
Get-ECSService -Cluster 'my-cluster'
Update-ECSService -Cluster 'my-cluster' -Service 'my-service' -DesiredCount 3
Get-ECSTask -Cluster 'my-cluster' -DesiredStatus RUNNING
```

---

## 5. CloudWatch และ Cost

```powershell
# CloudWatch metrics
Get-CWMetricData -MetricDataQuery @(
    @{
        Id         = 'cpu'
        MetricStat = @{
            Metric = @{Namespace='AWS/EC2'; MetricName='CPUUtilization';Dimensions=@(@{Name='InstanceId';Value='i-xxx'})}
            Period = 300
            Stat   = 'Average'
        }
        ReturnData = $true
    }
) -StartTime (Get-Date).AddHours(-1) -EndTime (Get-Date)

# Cost Explorer
$usage = Get-CECostAndUsage `
    -TimePeriod @{Start=(Get-Date -Format 'yyyy-MM-01'); End=(Get-Date -Format 'yyyy-MM-dd')} `
    -Granularity MONTHLY `
    -Metrics 'UnblendedCost' `
    -GroupBy @(@{Type='SERVICE'; Key='SERVICE'})

$usage.ResultsByTime[0].Groups | ForEach-Object {
    [PSCustomObject]@{
        Service = $_.Keys[0]
        Cost    = "$($_.Metrics.UnblendedCost.Amount) $($_.Metrics.UnblendedCost.Unit)"
    }
} | Sort-Object Cost -Descending | Format-Table
```

---

**ก่อนหน้า ← [Part 35](Part-35.md) | ต่อไป → [Part 37: PS Providers](Part-37.md)**
