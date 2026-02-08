# Cookbook: Windows Server Management & Troubleshooting

> [!NOTE]
> Practical guide to Windows Server administration using PowerShell.
> Covers: Service management, system monitoring, event logs, and common issues.

## 1. PowerShell Basics

### Check PowerShell Version
```powershell
$PSVersionTable
# Shows Version, PSEdition (Desktop/Core), etc.
```

### Execution Policy
```powershell
# Check current policy
Get-ExecutionPolicy

# Allow running scripts (for current user)
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser

# Or machine-wide
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope LocalMachine

# Bypass for one command
powershell -ExecutionPolicy Bypass -Command "& 'C:\script.ps1'"
```

## 2. Service Management

### View Services
```powershell
# List all services
Get-Service

# Get specific service
Get-Service -Name "ServiceName"

# Show running services only
Get-Service | Where-Object {$_.Status -eq 'Running'}

# Show stopped services
Get-Service | Where-Object {$_.Status -eq 'Stopped'}

# Get service details
Get-Service -Name "ServiceName" | Format-List *
```

### Control Services
```powershell
# Start service
Start-Service -Name "ServiceName"

# Stop service
Stop-Service -Name "ServiceName"

# Restart service
Restart-Service -Name "ServiceName"

# Force stop (if hung)
Stop-Service -Name "ServiceName" -Force

# Set startup type
Set-Service -Name "ServiceName" -StartupType Automatic
# Options: Automatic, Manual, Disabled

# Check if service is running
(Get-Service -Name "ServiceName").Status -eq 'Running'
```

## 3. Process Management

### View Processes
```powershell
# List all processes
Get-Process

# Get specific process
Get-Process -Name "processname"

# Sort by memory usage
Get-Process | Sort-Object -Property WS -Descending | Select-Object -First 10

# Find process using specific port
Get-NetTCPConnection -LocalPort 3000 | Select-Object OwningProcess
# Then: Get-Process -Id <PID>

# Get process details
Get-Process -Name "processname" | Format-List *
```

### Control Processes
```powershell
# Kill process by name
Stop-Process -Name "processname"

# Kill process by PID
Stop-Process -Id 1234

# Force kill
Stop-Process -Name "processname" -Force

# Show process command line
Get-CimInstance Win32_Process -Filter "name='cmd.exe'" | Select-Object ProcessId, CommandLine
```

## 4. System Monitoring

### CPU & Memory
```powershell
# CPU usage
Get-WmiObject win32_processor | Select-Object LoadPercentage

# Memory available
$mem = Get-WmiObject win32_operatingsystem
"Available: $([math]::round($mem.FreePhysicalMemory/1mb,2)) MB"
"Total: $([math]::round($mem.TotalVisibleMemorySize/1mb,2)) MB"

# Memory per process
Get-Process | Select-Object Name, @{Name="Memory(MB)";Expression={[math]::round($_.WS/1mb,2)}} | Sort-Object "Memory(MB)" -Descending

# CPU per process (top 10)
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10 Name, CPU
```

### Disk Space
```powershell
# Disk usage per drive
Get-Volume | Select-Object DriveLetter, FileSystemLabel, @{Name="Size(GB)";Expression={[math]::round($_.Size/1gb,2)}}, @{Name="Free(GB)";Expression={[math]::round($_.SizeRemaining/1gb,2)}}

# Disk usage by folder
$root = "C:\"
Get-ChildItem $root -Directory | ForEach-Object {
    $size = (Get-ChildItem $_.FullName -Recurse | Measure-Object -Property Length -Sum).Sum
    [PSCustomObject]@{
        Folder = $_.Name
        Size_GB = [math]::round($size/1gb,2)
    }
} | Sort-Object Size_GB -Descending

# Find large files
Get-ChildItem -Path "C:\Users\*" -Recurse -ErrorAction SilentlyContinue | Sort-Object Length -Descending | Select-Object -First 20 FullName, @{Name="Size(MB)";Expression={[math]::round($_.Length/1mb,2)}}
```

### Network
```powershell
# Network interfaces
Get-NetAdapter | Select-Object Name, Status, LinkSpeed

# IP configuration
Get-NetIPConfiguration

# Network statistics
Get-NetTCPConnection | Group-Object State | Select-Object Name, Count

# DNS configuration
Get-DnsClientServerAddress

# Ping
Test-Connection -ComputerName "example.com"
```

## 5. Event Logs

### View Logs
```powershell
# Recent errors from System log
Get-EventLog -LogName System -EntryType Error -Newest 10

# Recent warnings
Get-EventLog -LogName System -EntryType Warning -Newest 10

# Application log
Get-EventLog -LogName Application -Newest 10

# Security log (requires admin)
Get-EventLog -LogName Security -Newest 10

# Custom log from last hour
Get-EventLog -LogName System -After (Get-Date).AddHours(-1)
```

### Search Event Logs
```powershell
# Search by source
Get-EventLog -LogName System -Source "DNS Client Events"

# Search by event ID
Get-EventLog -LogName System -InstanceId 1001

# Search by message (like grep)
Get-EventLog -LogName Application | Where-Object {$_.Message -match "error"}

# Count events by type
Get-EventLog -LogName System -Newest 1000 | Group-Object -Property EntryType | Select-Object Name, Count
```

### Clear Logs
```powershell
# Clear event log (requires admin)
Clear-EventLog -LogName System

# Or use
Remove-Item 'HKLM:\System\CurrentControlSet\Services\EventLog\System\EventLog' -Force
```

## 6. Windows Firewall

### View Rules
```powershell
# List all firewall rules
Get-NetFirewallRule

# Show enabled rules
Get-NetFirewallRule -Enabled $true

# Get inbound rules
Get-NetFirewallRule -Direction Inbound

# Show specific rule details
Get-NetFirewallRule -DisplayName "Rule Name" | Get-NetFirewallPortFilter
```

### Manage Rules
```powershell
# Allow port
New-NetFirewallRule -DisplayName "Allow Port 3000" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 3000

# Block port
New-NetFirewallRule -DisplayName "Block Port 3000" -Direction Inbound -Action Block -Protocol TCP -LocalPort 3000

# Enable rule
Enable-NetFirewallRule -DisplayName "Rule Name"

# Disable rule
Disable-NetFirewallRule -DisplayName "Rule Name"

# Remove rule
Remove-NetFirewallRule -DisplayName "Rule Name"

# Allow application
New-NetFirewallRule -DisplayName "Allow App" -Direction Inbound -Action Allow -Program "C:\Program Files\App\app.exe"
```

### Firewall Profiles
```powershell
# Get firewall status
Get-NetFirewallProfile

# Enable firewall
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled $true

# Disable firewall (careful!)
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled $false
```

## 7. Scheduled Tasks

### View Tasks
```powershell
# List all scheduled tasks
Get-ScheduledTask

# Get specific task
Get-ScheduledTask -TaskName "TaskName"

# Get task details
Get-ScheduledTask -TaskName "TaskName" | Select-Object *

# Get task status
Get-ScheduledTask -TaskName "TaskName" | Select-Object State, LastRunTime, NextRunTime
```

### Create Tasks
```powershell
# Simple scheduled task
$action = New-ScheduledTaskAction -Execute "PowerShell.exe" -Argument "-NoProfile -WindowStyle Hidden -Command 'C:\scripts\backup.ps1'"
$trigger = New-ScheduledTaskTrigger -Weekly -DaysOfWeek Sunday -At 2am
Register-ScheduledTask -TaskName "Weekly Backup" -Action $action -Trigger $trigger -RunLevel Highest

# Run immediately
Start-ScheduledTask -TaskName "Weekly Backup"
```

### Delete Tasks
```powershell
# Unregister (delete) task
Unregister-ScheduledTask -TaskName "TaskName" -Confirm:$false
```

## 8. System Information

### OS & Hardware Info
```powershell
# OS version
Get-CimInstance Win32_OperatingSystem | Select-Object Caption, Version, BuildNumber

# Hostname
$env:COMPUTERNAME

# Uptime
$uptime = (Get-Date) - (Get-CimInstance Win32_OperatingSystem).LastBootUpTime
"Uptime: $($uptime.Days) days, $($uptime.Hours) hours, $($uptime.Minutes) minutes"

# Installed RAM
[math]::round((Get-CimInstance Win32_PhysicalMemory | Measure-Object -Property Capacity -Sum).Sum/1gb) | Out-String

# CPU info
Get-CimInstance Win32_Processor | Select-Object Name, Cores, LogicalProcessors, MaxClockSpeed
```

## 9. File Management

### Common Tasks
```powershell
# Create directory
New-Item -ItemType Directory -Path "C:\NewFolder" -Force

# Copy file
Copy-Item -Path "C:\source\file.txt" -Destination "C:\dest\file.txt" -Force

# Move file
Move-Item -Path "C:\source\file.txt" -Destination "C:\dest\file.txt" -Force

# Delete file
Remove-Item -Path "C:\path\file.txt" -Force

# Delete directory
Remove-Item -Path "C:\path\folder" -Recurse -Force

# List files recursively
Get-ChildItem -Path "C:\" -Recurse -Filter "*.log"

# Get file hash (checksum)
Get-FileHash -Path "C:\file.txt" -Algorithm SHA256
```

## 10. Common Troubleshooting

### Service Won't Start
```powershell
# 1. Check service status
Get-Service -Name "ServiceName" | Format-List *

# 2. Check dependencies
(Get-Service -Name "ServiceName").DependentServices

# 3. Check event log
Get-EventLog -LogName System -Source "Service Control Manager" -Newest 10

# 4. Restart dependent services first
Start-Service -Name "DependentService"
Start-Service -Name "ServiceName"
```

### High Memory Usage
```powershell
# 1. Find memory hog
Get-Process | Sort-Object WS -Descending | Select-Object -First 5 Name, @{Name="Memory(MB)";Expression={[math]::round($_.WS/1mb,2)}}

# 2. Check if legitimate
# If not, kill process
Stop-Process -Name "processname" -Force

# 3. Clear memory cache (if needed)
Clear-Host  # Just clears screen
# For actual memory, restart service or VM
```

### Port Already in Use
```powershell
# 1. Find process using port
$port = 3000
[System.Net.NetworkInformation.IPGlobalProperties]::GetIPGlobalProperties().GetActiveTcpConnections() |
    Where-Object {$_.LocalEndPoint.Port -eq $port} |
    ForEach-Object {
        Get-Process -Id $_.OwningProcess | Select-Object Name, Id
    }

# 2. Kill process
Stop-Process -Id <PID> -Force
```

### Disk Full
```powershell
# 1. Check disk usage
Get-Volume | Format-Table DriveLetter, FileSystemLabel, SizeRemaining

# 2. Find large files
Get-ChildItem -Path "C:\" -Recurse -ErrorAction SilentlyContinue | Sort-Object Length -Descending | Select-Object -First 10 FullName, @{Name="Size(MB)";Expression={[math]::round($_.Length/1mb,2)}}

# 3. Check temp folders
Get-ChildItem "C:\Windows\Temp" -Recurse | Measure-Object -Property Length -Sum

# 4. Clean temp files
Remove-Item "C:\Windows\Temp\*" -Recurse -Force -ErrorAction SilentlyContinue

# 5. Clean Windows cache
Cleanmgr.exe
```

## 11. Useful Scripts

### System Health Check
```powershell
# Quick health check
Write-Host "=== SYSTEM HEALTH CHECK ===" -ForegroundColor Green

# CPU
$cpu = Get-WmiObject win32_processor | Select-Object LoadPercentage
Write-Host "CPU Usage: $($cpu.LoadPercentage)%"

# Memory
$mem = Get-WmiObject win32_operatingsystem
$freeGB = [math]::round($mem.FreePhysicalMemory/1mb/1024, 2)
$totalGB = [math]::round($mem.TotalVisibleMemorySize/1mb/1024, 2)
Write-Host "Memory: $freeGB GB / $totalGB GB free"

# Disk
Get-Volume | Where-Object {$_.DriveType -eq "Fixed"} | ForEach-Object {
    $freeGB = [math]::round($_.SizeRemaining/1gb,2)
    $sizeGB = [math]::round($_.Size/1gb,2)
    Write-Host "Disk $($_.DriveLetter): $freeGB GB / $sizeGB GB free"
}

# Services with errors
Write-Host "Services with errors:" -ForegroundColor Yellow
Get-EventLog -LogName System -EntryType Error -Newest 5 | Select-Object TimeGenerated, Source, Message
```

## 12. Common Aliases & Functions

### Add to PowerShell Profile
```powershell
# Get profile location
$PROFILE

# Edit profile (create if doesn't exist)
notepad $PROFILE

# Add shortcuts
function ll { Get-ChildItem -Force $args }
function rr { Restart-Service -Name $args -Force }
function status { Get-Service -Name $args | Format-List * }
function mem { Get-Process | Sort-Object WS -Descending | Select-Object -First 10 Name, @{Name="Memory(MB)";Expression={[math]::round($_.WS/1mb,2)}} }
function ports { Get-NetTCPConnection -State Listen | Select-Object LocalAddress, LocalPort, OwningProcess }
```

---

**PowerShell Best Practices**:
- Always run as Administrator for system tasks
- Use `-WhatIf` before destructive commands
- Wrap commands in `try/catch` blocks
- Use `Write-Host` for user output, `Write-Output` for pipeline
- Set execution policy appropriately
- Use `-Force` carefully on destructive operations
- Quote paths with spaces
- Use `Get-Help` before running unfamiliar commands
