# Part 60: Kubernetes & Container Orchestration

> **ระดับ**: 🔴 Advanced | **เวลา**: ~5 ชั่วโมง

---

## 1. kubectl Wrapper

```powershell
function Get-K8sPods {
    param([string]$Namespace = 'default', [string]$Label)
    $args = @('get','pods','-n',$Namespace,'-o','json')
    if ($Label) { $args += @('-l',$Label) }
    $json = kubectl @args | ConvertFrom-Json
    $json.items | Select-Object `
        @{n='Name';    e={$_.metadata.name}},
        @{n='Status';  e={$_.status.phase}},
        @{n='Ready';   e={"$(@($_.status.containerStatuses | Where-Object ready).Count)/$($_.spec.containers.Count)"}},
        @{n='Restarts';e={($_.status.containerStatuses | Measure-Object restartCount -Sum).Sum}},
        @{n='Age';     e={(([datetime]::Now) - [datetime]$_.metadata.creationTimestamp).ToString('dd\.hh\:mm')}}
}

function Scale-K8sDeployment {
    param([string]$Name, [string]$Namespace='default', [int]$Replicas)
    kubectl scale deployment/$Name -n $Namespace --replicas=$Replicas
    Write-Host "Scaled $Name to $Replicas replicas" -ForegroundColor Green
}

function Restart-K8sDeployment {
    param([string]$Name, [string]$Namespace='default')
    kubectl rollout restart deployment/$Name -n $Namespace
    kubectl rollout status deployment/$Name -n $Namespace --timeout=300s
}
```

---

## 2. Deployment Automation

```powershell
function Deploy-K8sApp {
    param(
        [string]$AppName,
        [string]$Image,
        [string]$Namespace = 'default',
        [int]$Replicas      = 2,
        [hashtable]$EnvVars = @{}
    )
    
    $envYaml = ($EnvVars.GetEnumerator() | ForEach-Object {
        "        - name: $($_.Key)`n          value: `"$($_.Value)`""
    }) -join "`n"
    
    $manifest = @"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: $AppName
  namespace: $Namespace
spec:
  replicas: $Replicas
  selector:
    matchLabels:
      app: $AppName
  template:
    metadata:
      labels:
        app: $AppName
    spec:
      containers:
      - name: $AppName
        image: $Image
        ports:
        - containerPort: 8080
        env:
$envYaml
        resources:
          requests: { memory: "128Mi", cpu: "100m" }
          limits:   { memory: "256Mi", cpu: "500m" }
        readinessProbe:
          httpGet: { path: /health, port: 8080 }
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: $AppName
  namespace: $Namespace
spec:
  selector:
    app: $AppName
  ports:
  - port: 80
    targetPort: 8080
"@
    
    $tmpFile = [IO.Path]::GetTempFileName() + '.yaml'
    $manifest | Set-Content $tmpFile
    kubectl apply -f $tmpFile -n $Namespace
    Remove-Item $tmpFile
    
    kubectl rollout status deployment/$AppName -n $Namespace --timeout=120s
    Get-K8sPods -Namespace $Namespace -Label "app=$AppName" | Format-Table
}

Deploy-K8sApp -AppName 'my-api' -Image 'myregistry.azurecr.io/my-api:v1.2.0' -Namespace 'production' -Replicas 3
```

---

## 3. Helm Chart Management

```powershell
function Get-HelmReleases {
    param([string]$Namespace = '')
    $args = @('list','--output','json')
    if ($Namespace) { $args += @('-n',$Namespace) } else { $args += '-A' }
    helm @args | ConvertFrom-Json
}

function Install-HelmChart {
    param(
        [string]$ReleaseName,
        [string]$Chart,
        [string]$Namespace   = 'default',
        [string]$ValuesFile,
        [hashtable]$SetValues = @{}
    )
    $args = @('upgrade','--install',$ReleaseName,$Chart,'-n',$Namespace,'--create-namespace','--wait')
    if ($ValuesFile) { $args += @('-f',$ValuesFile) }
    foreach ($kv in $SetValues.GetEnumerator()) { $args += @('--set',"$($kv.Key)=$($kv.Value)") }
    helm @args
}

# Deploy nginx ingress
Install-HelmChart -ReleaseName 'ingress-nginx' -Chart 'ingress-nginx/ingress-nginx' `
    -Namespace 'ingress-nginx' `
    -SetValues @{ 'controller.replicaCount'='2'; 'controller.service.type'='LoadBalancer' }

# Deploy cert-manager
Install-HelmChart -ReleaseName 'cert-manager' -Chart 'jetstack/cert-manager' `
    -Namespace 'cert-manager' -SetValues @{ 'installCRDs'='true' }
```

---

## 4. Cluster Health Dashboard

```powershell
function Get-K8sClusterHealth {
    param([string]$Namespace = 'default')
    
    $nodes = (kubectl get nodes -o json | ConvertFrom-Json).items | ForEach-Object {
        $ready = ($_.status.conditions | Where-Object { $_.type -eq 'Ready' }).status
        [PSCustomObject]@{
            Node   = $_.metadata.name
            Status = $ready
            CPU    = $_.status.capacity.cpu
            MemGB  = [math]::Round([int64]($_.status.capacity.memory -replace 'Ki') / 1MB, 1)
        }
    }
    
    $pods       = (kubectl get pods -n $Namespace -o json | ConvertFrom-Json).items
    $failing    = $pods | Where-Object { $_.status.phase -notin @('Running','Succeeded') }
    
    [PSCustomObject]@{
        Nodes       = $nodes
        FailingPods = $failing.Count
        TotalPods   = $pods.Count
    }
}

$health = Get-K8sClusterHealth
$health.Nodes | Format-Table
Write-Host "Failing Pods: $($health.FailingPods) / $($health.TotalPods)" `
    -ForegroundColor $(if ($health.FailingPods -gt 0) { 'Red' } else { 'Green' })
```

---

**ก่อนหน้า ← [Part 59](Part-59.md) | ต่อไป → [Part 61: Terraform & IaC](Part-61.md)**
