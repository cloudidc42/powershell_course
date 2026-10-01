# Part 72: GUI Applications ด้วย PowerShell

> **ระดับ**: 🟡 Professional | **เวลา**: ~4 ชั่วโมง

---

## 1. WinForms Application

```powershell
Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName System.Drawing

# System Monitor GUI
function New-SystemMonitorUI {
    $form = [System.Windows.Forms.Form]@{
        Text            = 'System Monitor'
        Width           = 800
        Height          = 500
        StartPosition   = 'CenterScreen'
        BackColor       = [System.Drawing.Color]::FromArgb(30,30,30)
        ForeColor       = [System.Drawing.Color]::White
    }
    
    # Title label
    $title = [System.Windows.Forms.Label]@{
        Text     = 'System Monitor'
        Font     = [System.Drawing.Font]::new('Segoe UI', 16, [System.Drawing.FontStyle]::Bold)
        ForeColor= [System.Drawing.Color]::CornflowerBlue
        Location = [System.Drawing.Point]::new(20, 10)
        AutoSize = $true
    }
    $form.Controls.Add($title)
    
    # CPU Progress Bar
    $cpuLabel = [System.Windows.Forms.Label]@{
        Text='CPU: 0%'; Location=[System.Drawing.Point]::new(20,60); AutoSize=$true; ForeColor=[System.Drawing.Color]::White
    }
    $cpuBar = [System.Windows.Forms.ProgressBar]@{
        Location=[System.Drawing.Point]::new(20,85); Width=750; Height=25; Minimum=0; Maximum=100
    }
    $form.Controls.AddRange(@($cpuLabel, $cpuBar))
    
    # RAM Progress Bar
    $ramLabel = [System.Windows.Forms.Label]@{
        Text='RAM: 0%'; Location=[System.Drawing.Point]::new(20,120); AutoSize=$true; ForeColor=[System.Drawing.Color]::White
    }
    $ramBar = [System.Windows.Forms.ProgressBar]@{
        Location=[System.Drawing.Point]::new(20,145); Width=750; Height=25; Minimum=0; Maximum=100
    }
    $form.Controls.AddRange(@($ramLabel, $ramBar))
    
    # Process List
    $listView = [System.Windows.Forms.ListView]@{
        Location = [System.Drawing.Point]::new(20, 190)
        Width    = 750; Height = 250
        View     = [System.Windows.Forms.View]::Details
        FullRowSelect = $true
        GridLines     = $true
        BackColor     = [System.Drawing.Color]::FromArgb(45,45,48)
        ForeColor     = [System.Drawing.Color]::White
    }
    @('Process','PID','CPU','Memory (MB)') | ForEach-Object {
        $col = [System.Windows.Forms.ColumnHeader]@{ Text=$_; Width=180 }
        $listView.Columns.Add($col) | Out-Null
    }
    $form.Controls.Add($listView)
    
    # Update timer
    $timer = [System.Windows.Forms.Timer]@{ Interval = 2000; Enabled = $true }
    $timer.Add_Tick({
        # CPU
        $cpu = [int](Get-Counter '\Processor(_Total)\% Processor Time').CounterSamples.CookedValue
        $cpuLabel.Text = "CPU: $cpu%"
        $cpuBar.Value  = [Math]::Min($cpu, 100)
        $cpuBar.ForeColor = if ($cpu -gt 80) { [Drawing.Color]::Red } elseif ($cpu -gt 60) { [Drawing.Color]::Orange } else { [Drawing.Color]::Green }
        
        # RAM
        $os     = Get-WmiObject Win32_OperatingSystem
        $ramPct = [int](100 - ($os.FreePhysicalMemory / $os.TotalVisibleMemorySize * 100))
        $ramLabel.Text = "RAM: $ramPct% (Free: $([math]::Round($os.FreePhysicalMemory/1MB, 1)) GB)"
        $ramBar.Value  = [Math]::Min($ramPct, 100)
        
        # Top processes
        $listView.BeginUpdate()
        $listView.Items.Clear()
        Get-Process | Sort-Object CPU -Descending | Select-Object -First 20 | ForEach-Object {
            $item = [System.Windows.Forms.ListViewItem]::new($_.Name)
            $item.SubItems.Add($_.Id.ToString()) | Out-Null
            $item.SubItems.Add([math]::Round($_.CPU, 1).ToString()) | Out-Null
            $item.SubItems.Add([math]::Round($_.WorkingSet64/1MB, 1).ToString()) | Out-Null
            $listView.Items.Add($item) | Out-Null
        }
        $listView.EndUpdate()
    })
    
    $form.Add_Shown({ $timer.Start() })
    $form.Add_FormClosing({ $timer.Stop(); $timer.Dispose() })
    
    [System.Windows.Forms.Application]::Run($form)
}

New-SystemMonitorUI
```

---

## 2. WPF Application

```powershell
Add-Type -AssemblyName PresentationFramework
Add-Type -AssemblyName PresentationCore

$xaml = @'
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        Title="PowerShell Tool" Width="600" Height="400">
    <Grid Background="#1E1E1E">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>
        <TextBox Grid.Row="0" x:Name="cmdInput" Background="#2D2D2D" Foreground="White"
                 FontFamily="Consolas" FontSize="14" Margin="10"
                 AcceptsReturn="False" Padding="5"/>
        <ListBox Grid.Row="1" x:Name="outputList" Background="#1E1E1E"
                 Foreground="#00FF00" FontFamily="Consolas" Margin="10"/>
        <Button  Grid.Row="2" x:Name="runBtn" Content="Run" Margin="10"
                 Background="#007ACC" Foreground="White" Height="35"/>
    </Grid>
</Window>
'@

$reader  = [System.Xml.XmlReader]::Create([System.IO.StringReader]::new($xaml))
$window  = [System.Windows.Markup.XamlReader]::Load($reader)

$cmdInput  = $window.FindName('cmdInput')
$outputList= $window.FindName('outputList')
$runBtn    = $window.FindName('runBtn')

$runBtn.Add_Click({
    $cmd = $cmdInput.Text.Trim()
    if (-not $cmd) { return }
    
    $outputList.Items.Clear()
    try {
        $results = Invoke-Expression $cmd 2>&1
        foreach ($line in $results) {
            $outputList.Items.Add($line.ToString()) | Out-Null
        }
    } catch {
        $outputList.Items.Add("ERROR: $_") | Out-Null
    }
})

$window.ShowDialog() | Out-Null
```

---

## 3. Notification & System Tray

```powershell
Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName System.Drawing

# Toast notification
function Show-Notification {
    param([string]$Title, [string]$Message, [string]$Icon = 'Info')
    
    $balloon = [System.Windows.Forms.NotifyIcon]@{
        Icon    = [System.Drawing.SystemIcons]::Information
        Visible = $true
    }
    $balloon.ShowBalloonTip(
        5000,
        $Title,
        $Message,
        [System.Windows.Forms.ToolTipIcon]::$Icon
    )
    Start-Sleep 5
    $balloon.Dispose()
}

Show-Notification -Title 'Deploy Complete' -Message 'v2.1.0 deployed successfully!' -Icon 'Info'

# System tray app
function New-SystemTrayApp {
    param([string]$AppName, [hashtable]$MenuItems)
    
    $notifyIcon = [System.Windows.Forms.NotifyIcon]@{
        Icon    = [System.Drawing.SystemIcons]::Application
        Text    = $AppName
        Visible = $true
    }
    
    $contextMenu = [System.Windows.Forms.ContextMenuStrip]::new()
    
    foreach ($item in $MenuItems.GetEnumerator()) {
        $menuItem = [System.Windows.Forms.ToolStripMenuItem]::new($item.Key)
        $action   = $item.Value  # capture
        $menuItem.Add_Click({ & $action })
        $contextMenu.Items.Add($menuItem) | Out-Null
    }
    
    $exitItem = [System.Windows.Forms.ToolStripMenuItem]::new('Exit')
    $exitItem.Add_Click({
        $notifyIcon.Visible = $false
        $notifyIcon.Dispose()
        [System.Windows.Forms.Application]::Exit()
    })
    $contextMenu.Items.Add($exitItem) | Out-Null
    $notifyIcon.ContextMenuStrip = $contextMenu
    
    [System.Windows.Forms.Application]::Run()
}
```

---

**ก่อนหน้า ← [Part 71](Part-71.md) | ต่อไป → [Part 73: Web Scraping](Part-73.md)**
